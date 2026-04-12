# A Pattern for Sharing Data Between Websockets in Haskell

I'd like to share a wee pattern I've found useful for writing simple WebSocket apps in Haskell - specifically for [wordify](https://wordify.gordo.life) - a ~ crossword board game ~ - a bit like https://woogles.io/ .

I have in mind websocket applications where there's some sort of 'resource' that is shared between multiple websocket clients that need to be kept updated such as:

* A 'Game' resource for a turn based game such as a Chess, Draughts (that's "checkers" to some of you) or Noughts and Crosses (that's "tic-tac-toe" to some of you) that's updated on each new move with the game state and then Websockets are sent a message about what just changed.
* A 'Chat' resource where the shared state keeps track of users currently in the room and their connectivity status

For simplicity, we'll implement a multiplayer Noughts and Crosses (as we'll call it in a parochially defiant act of patriotism against the US-centric internet :P) server as our example.

## The Core Game Server 

The canonical examples on Yesod WebSockets illustate how to use a shared broadcast TChan as the basis for a chat server (blog [here](https://www.yesodweb.com/blog/2014/03/wai-yesod-websockets), code example (here)[https://github.com/yesodweb/yesod/blob/master/yesod-websockets/chat.hs].) 

This shared TChan broadcast channel is a useful abstraction which will form part of our implementation but unlike the materials above we'll focus on a pattern for having multiple games active at one time. 

A naive approach might be to plonk our games in a `Map GameId GameState` type inside a mutable variable using (TVar)[https://hackage.haskell.org/package/stm-2.4.2/docs/Control-Concurrent-STM-TVar.html] that's populated with a given game when at least one WebSocket needs access to the game to play moves or spectate it. Here we'll set up our `GameState` types including our board state, our two players, channel to broadcast moves, etc as well as our game map:

```
type GameId = T.Text
data Player = Player { username ::T.Text, userId:: T.Text }
type ConnectionCount = Int

-- A board square is either empty, an X or an O
data Square = Empty | X | O

-- For simplicity, a board is just a list of 9 squares - we flatten it out to a 1 dimensional representation
type Board = [Square] 

data GameUpdate = NewBoard Board

data GameState = GameState {
    board:: TVar Board,
    player1 :: Player,
    player2 :: Player,
    -- The channel will be updated each time a move is made to notify listening websockets
    gameChan :: (TChan GameUpdate),

    -- The number of connected websockets to this game
    gameConnections :: TVar ConnectionCount
}

data GameMap = TVar (Map GameId GameState)
```

Since we're using a TVar each WebSocket thread can load the game into the game map if necessary by running `modifyTVar` to update the map.

That just won't do though! Only one thread can update the map entries at a time, and we have aspirations of being the Internet's go-to Noughts and Crosses server! We'll need something that scales better than that while we've still got the luxury of vertically scaling one machine along our path to world domination. (stm-containers)[https://nikita-volkov.github.io/stm-containers/] to the rescue! It uses a Hash Array Mapped Trie datastructure to split the map into chunks that can be updated inside independent STM transactions allowing for more concurrency.

```
import qualified StmContainers.Map as M
data GameMap = M.Map GameId GameState
```

Our next challenge is to load the game state into the game map when the first websocket connects to subscribe to and update the game, and remove it from the map when it's no longer needed so that we don't run out of memory in our path to world domination.

## Loading the Game

Imagine our Yesod 'App' type had a GameRepository that was a data store we could use to access our persisted TicTacToe games:

```
data App = App { gameRepository :: GameRepository }
```

```
data GameEntity = GameEntity {
    gameEntityBoard :: Board,
    gameEntityPlayer1 :: Player,
    gameEntityPlayer2 :: Player
}

loadGame :: GameRepository -> GameId -> IO (Either Text GameEntity)
```

We will leverage this function to load the game into the cache when it's not already populated.

We will use (Software Transactional Memory (STM))[https://academy.fpblock.com/haskell/library/stm/] abstraction to ensure that if two websocket connections try to insert an entry into the game map at the same time, they both end up with the same 'GameState' instance so that they receive game updates on the broadcast channel.

```
getGameState :: GameRepository -> GameMap -> GameId -> IO (Either Text GameState)
getGameState gameRepository gameMap gameId =
    -- If getting the game from the cache fails we try loading it into the cache instead using the <|> (alternative) function
    runExceptT $ mapExceptT atomically getCachedEntry <|> loadCachedEntry
  where
    getCachedEntry :: ExceptT Text STM GameState
    getCachedEntry = ExceptT $ do
        cached <- M.lookup gameId gameMap
        case cached of
            Just gameState -> Right <$> registerSharer gameState
            -- Using mempty here means an error returned from 'loadCachedEntry' will take precedence over this error due to the semantics of
            -- the <|> function (arguably a bit hacky :P)
            Nothing -> return $ Left mempty

    loadCachedEntry :: ExceptT Text IO GameState
    loadCachedEntry = do
        entity <- ExceptT $ loadGame gameRepository gameId
        freshState <- liftIO $ mapGameState entity
        -- It's important we recheck in the same transaction as writing the entry to the cache
        -- that another thread hasn't inserted it into the cache in the meantime so that we 
        -- can make sure we're sharing the same TVars and channels, etc
        mapExceptT atomically $ getCachedEntry <|> lift (insertFresh freshState)

    insertFresh :: GameState -> STM GameState
    insertFresh freshState = do
        M.insert freshState gameId gameMap
        registerSharer freshState

    registerSharer :: GameState -> STM GameState
    registerSharer gameState = do
        modifyTVar' (gameConnections gameState) (+ 1)
        return gameState

    mapGameState :: GameEntity -> IO GameState
    mapGameState GameEntity { gameEntityBohttps://hackage.haskell.org/package/shared-resource-cacheard = b, gameEntityPlayer1 = p1, gameEntityPlayer2 = p2 } = do
        boardVar <- newTVarIO b
        chan <- newBroadcastTChanIO
        connVar <- newTVarIO 0
        return GameState
            { board = boardVar
            , player1 = p1
            , player2 = p2
            , gameChan = chan
            , gameConnections = connVar
            }
```

In our `loadCachedEntry` function, in an STM transaction we check if the item is already in the cache (via `getCachedEntry`) and if it is not we insert it and return the newly inserted game entry.

This means that if there is a conflict in the middle of the transaction because another websocket thread has updated our cache in another parallel transaction, the transaction will be retried and both threads will end up with the same 'gamestate' entry and see the same updates sent via the shared broadcast channel, etc.


## Taming the Thundering Herd

There's a small improvement we can make to make sure that multiple threads don't try to load our game at the same time, wasting trips to the database just to throw the result away. This could happen in a thundering herd of observers to watch the latest tournament game between Noughts and Crosses grandmasters!

We will use an (MVar)[https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Control-Concurrent-MVar.html] to allow a thread to lay claim to responsibility for loading the resource and signal any other threads waiting when it has done so. 

The game entry in our game map will now either be a game loaded into the cache or an MVar that can be awaited (blocked on) signalling that the game has been loaded into the cache:

```
data GameEntry = LoadedGame GameState | LoadingGame (MVar ())
type GameMap = M.Map GameId GameEntry
```

When we first try to retrieve a game, we have to decide the action we need to take based on the cache state, which we represent with this type:

```
data CacheDecision
    -- The game is already in the cache
    = Hit GameState
    -- We're awaiting another thread loading the game
    | Await (MVar ())
    -- We have claimed ownership over loading the game
    | Claimed (MVar ())
```

When we load a game we transactionally lay claim to loading the resource if it's not already loaded or is currently being loaded by writing a 'LoadingGame' entry to cache:

```
getGameState :: GameRepository -> GameMap -> GameId -> IO (Either Text GameState)
getGameState gameRepository gameMap gameId = do
    pendingMvar <- newEmptyMVar
    decision <- atomically $ do
        cached <- M.lookup gameId gameMap
        case cached of
            Just (LoadedGame gameState) ->
                Hit <$> registerSharer gameState
            Just (LoadingGame existingMvar) ->
                return $ Await existingMvar
            Nothing -> do
                M.insert (LoadingGame pendingMvar) gameId gameMap
                return $ Claimed pendingMvar
    case decision of
        Hit gameState ->
            return $ Right gameState
        Await existingMvar -> do
            -- Once signalled that the game has been loaded, we recursively start again
            readMVar existingMvar
            getGameState gameRepository gameMap gameId
        Claimed ownedMvar ->
            loadAndPublish ownedMvar
```

This is performed in an STM transaction, so if multiple threads try to insert their 'MVar' claiming ownership then the losers of the race will retry the transaction and see the winning thread's claim. They will then block on it being populated by the other thread using using (readMVar)[https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Control-Concurrent-MVar.html#v:readMVar], which serves as a signal that they can now load the game from the cache.

When loading the resource we must make sure that threads don't wait on the MVar indefinetly if an error is thrown while retrieving the game:

```
    loadAndPublish :: MVar () -> IO (Either Text GameState)
    loadAndPublish ownedMvar = do
        attempted <- tryLoad
        result <- case attempted of
            Left ioErr -> do
                -- An exception was thrown in the IO action. We remove the claim
                -- to allow another threat to attempt to load it
                atomically $ M.delete gameId gameMap
                return $ Left (T.pack (show ioErr))
            Right (Left err) -> do
                -- The action returned a 'Left.' We remove the claim to allow another thread
                -- to attempt to load it
                atomically $ M.delete gameId gameMap
                return $ Left err
            Right (Right entity) -> do
                freshState <- mapGameState entity
                atomically $ do
                    -- Replace the 'Loading' cache entry with the loaded game entry
                    M.insert (LoadedGame freshState) gameId gameMap
                    _ <- registerSharer freshState
                    return ()
                return $ Right freshState
        -- Wake up any threads currently waiting on our LoadingGame placeholder.
        putMVar ownedMvar ()
        return result
      where
        tryLoad :: IO (Either IOException (Either Text GameEntity))
        tryLoad = try (loadGame gameRepository gameId)
```

# Cache Expiry

Our last challenge is to make sure that entries are removed from the cache when it's safe to do so (there are no websocket subscriptions.) Our Noughts and Crosses server will be very busy with many games being started and ended and we can't have ourselves running out of heap space due to a memory leak of games remaining in the cache!

When we think of garaunteeing the safe release of scarce resources, one of the options our mind turns to the (resourcet)[https://hackage.haskell.org/package/resourcet] package.

We will use the resourcet package's (allocate)[https://hackage.haskell.org/package/resourcet-1.3.0/docs/Control-Monad-Trans-Resource.html#v:allocate] function to register the action that must be taken when the game resource is no longer in scope (due to the websocket disconnecting or an exception being thrown.)

We have the 'gameConnections' variable already tracking the number of websocket handlers that are subscribed to the game. On connection, we will increment it as usual and on disconnection if the connection count has reached 0 we will remove it from the cache:

```
getGameState :: MonadResource m => GameRepository -> GameMap -> GameId -> m (Either Text GameState)
getGameState gameRepository gameMap gameId = do
    (_releaseKey, result) <- allocate acquireGameState releaseGameState
    return result
  where
    acquireGameState :: IO (Either Text GameState)
    acquireGameState = do
        pendingMvar <- newEmptyMVar
        decision <- atomically $ do
            cached <- M.lookup gameId gameMap
            case cached of
                Just (LoadedGame gameState) ->
                    Hit <$> registerSharer gameState
                Just (LoadingGame existingMvar) ->
                    return $ Await existingMvar
                Nothing -> do
                    M.insert (LoadingGame pendingMvar) gameId gameMap
                    return $ Claimed pendingMvar
        case decision of
            Hit gameState ->
                return $ Right gameState
            Await existingMvar -> do
                readMVar existingMvar
                acquireGameState
            Claimed ownedMvar ->
                loadAndPublish ownedMvar

    releaseGameState :: Either Text GameState -> IO ()
    releaseGameState (Left _) =
        -- Acquire reported a loading error and already cleaned up its placeholder,
        -- so there's no sharer count to decrement.
        return ()
    releaseGameState (Right gameState) = atomically $ do
        modifyTVar' (gameConnections gameState) (subtract 1)
        remaining <- readTVar (gameConnections gameState)
        when (remaining <= 0) $ M.delete gameId gameMap
```

Again, the STM abstraction has served us well: if a new websocket connection retrieves the game from the cache at the same time as it is being removed in another transaction, either:
* The transaction removing the cache entry is aborted, meaning the websocket doesn't end up with an orphaned game state and channel it will never get updates on
* The transaction retrieving the item from the cache is aborted, meaning the websocket thread has to insert it threshly

We can now use this game 'resource' in our websocket handler inside `runResourceT` meaning that the cache will be garaunteed to be cleaned up if the websocket was the last 'subscriber' on the websocket handler ending due to disconnection, etc:

```

```

If you think you might find this pattern for sharing state between websockets (or other situations) useful, I've published it as a library named (shared-resource-cache)[https://hackage.haskell.org/package/shared-resource-cache]. Source (here)[https://github.com/Happy0/shared-resource-cache]. 

*Note*: it works slightly different in that even when there are no 'sharers' of the given resource, it is not cleared from the cache until it has 'timed out' according to the passed in configuration.

