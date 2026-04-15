---
layout: post
title: "Writing a Turn Based Game Websocket Server in Haskell"
date: 2026-04-12
---

I'd like to share a wee pattern I've found useful for writing simple WebSocket apps in Haskell - specifically for my side project [wordify](https://wordify.gordo.life) - an [open source](https://github.com/Happy0/wordify-webapp/) ~ multiplayer crossword board game ~ .

I have in mind websocket applications where there's some sort of 'resource' that is shared between multiple websocket clients that need to be kept updated as the state changes such as:

* A 'Game' resource for a turn based game such as a Chess or Noughts and Crosses (that's "tic-tac-toe" to some of you) where we keep track of the game state.
* A 'Chat' resource where we keep track of things like the users in the room and whether they are currently online, etc

For simplicity, we'll implement a multiplayer server for Noughts and Crosses (as we'll call it in a parochially defiant act of patriotism against the US-centric internet :P) to illustrate the pattern.

I'll be exploring a pattern where we maintain a Map of all the Noughts and Crosses games we have in progress safely such that all the connected websockets see the same game updates.

## The Core Game Server 

There's a few different options for writing web apps in haskell. Yesod is the one I'm most familiar with. The canonical examples on Yesod WebSockets illustate how to use a shared broadcast [TChan](https://hackage.haskell.org/package/stm-2.5.3.1/docs/Control-Concurrent-STM-TChan.html) as the basis for a chat server (blog [here](https://www.yesodweb.com/blog/2014/03/wai-yesod-websockets), code example [here](https://github.com/yesodweb/yesod/blob/master/yesod-websockets/chat.hs).))

```haskell
data App = App (TChan Text)

chatApp :: WebSocketsT Handler ()
chatApp = do
    sendTextData ("Welcome to the chat server, please enter your name." :: Text)
    name <- receiveData
    sendTextData $ "Welcome, " <> name
    App writeChan <- getYesod
    readChan <- atomically $ do
        writeTChan writeChan $ name <> " has joined the chat"
        dupTChan writeChan
    (e :: Either SomeException ()) <- try $ race_
        (forever $ atomically (readTChan readChan) >>= sendTextData)
        (sourceWS $$ mapM_C (\msg ->
            atomically $ writeTChan writeChan $ name <> ": " <> msg))

    atomically $ case e of
        Left _ -> writeTChan writeChan $ name <> " has left the chat"
        Right () -> return ()

main :: IO ()
main = do
    chan <- atomically newBroadcastTChan
    warp 3000 $ App chan
```

In these examples, a global channel is accessed via `getYesod` which holds the channel state. It is duplicated by each websocket to subscribe to new messages in the chatroom.

This shared TChan broadcast channel is a useful abstraction which will form part of our implementation but unlike the materials above we'll need to maintain multiple gmaes at once with their own channel. 

A naive approach might be to plonk our games in a `Map GameId GameState` stored in a [TVar](https://hackage.haskell.org/package/stm-2.4.2/docs/Control-Concurrent-STM-TVar.html). A TVar is a mutable variable that can be updated by multiple threads (or websocket handlers) and can be updated transactionally along with other TVars (which we'll get into later.) 

```haskell
import Control.Concurrent.STM (atomically)
import Control.Concurrent.STM.TVar (TVar, newTVarIO)
import Data.Map.Strict (Map)
import qualified Data.Map.Strict as Map

data GameMap = TVar (Map GameId GameState)
data App = App GameMap

main :: IO ()
main = do
    gameMap <- newTVarIO Map.empty
    warp 3000 $ App gameMap
```


We'll also set up our `GameState` types including our board state, our two players, channel to broadcast move updates, etc:

```haskell
type GameId = T.Text
data Player = Player { username ::T.Text, userId:: T.Text }
type ConnectionCount = Int

-- A board square is either empty, an X or an O
data Square = Empty | X | O deriving (Eq)

instance ToJSON Square where
    toJSON Empty = toJSON ("" :: Text)
    toJSON X = toJSON ("X" :: Text)
    toJSON O = toJSON ("O" :: Text)

-- For simplicity, a board is just a list of 9 squares - we flatten it out to a 1 dimensional representation
type Board = [Square] 

data GameUpdate = NewBoard Board

data GameState = GameState {
    board:: TVar Board,
    player1 :: Player,
    player2 :: Player,
    -- The channel will be updated each time a move is made to notify listening websockets
    gameChan :: TChan GameUpdate,

    -- The number of connected websockets to this game
    gameConnections :: TVar ConnectionCount
}

data GameMap = TVar (Map GameId GameState)
```

Since we're using a TVar each WebSocket thread can load a new game into the game map if necessary by running [modifyTVar](https://hackage.haskell.org/package/stm-2.5.3.1/docs/Control-Concurrent-STM-TVar.html#v:modifyTVar) to update the map.

That just won't do though! Only one thread can update the map entries at a time, and we have aspirations of being the Internet's go-to Noughts and Crosses server! We'll need something that scales better than that while we've still got the luxury of vertically scaling one machine along our path to world domination. 

[stm-containers](https://nikita-volkov.github.io/stm-containers/) to the rescue! It uses a Hash Array Mapped Trie datastructure to split the map into chunks that can be updated inside independent STM transactions allowing for more concurrency.

```haskell
import qualified StmContainers.Map as M
data GameMap = M.Map GameId GameState
```

Our next challenge is to load the game state into the game map when the first websocket connects and remove it from the map when it's no longer needed so that we don't run out of memory in our path to world domination.

## Loading the Game

Imagine our Yesod 'App' type also had a GameRepository that was a data store we could use to access and persist our TicTacToe games:

```haskell
data App = App { gameRepository :: GameRepository, gameMap :: GameMap }
```

```haskell
data GameEntity = GameEntity {
    gameEntityBoard :: Board,
    gameEntityPlayer1 :: Player,
    gameEntityPlayer2 :: Player
}

loadGame :: GameRepository -> GameId -> IO (Either Text GameEntity)
```

We will leverage this function to load the game into the cache when it's not already populated.

We will use [Software Transactional Memory (STM)](https://academy.fpblock.com/haskell/library/stm/) abstraction to ensure that if two websocket connections try to insert an entry into the game map at the same time, they both end up with the same 'GameState' instance so that they receive game updates on the broadcast channel.

```haskell
getGameState :: GameRepository -> GameMap -> GameId -> IO (Either Text GameState)
getGameState gameRepository gameMap gameId = do
    existing <- atomically $ do
        cached <- M.lookup gameId gameMap
        case cached of
            Just gameState -> Just <$> registerSharer gameState
            Nothing -> return Nothing
    case existing of
        Just gameState ->
            return $ Right gameState
        Nothing -> do
            -- Note: there are ways we could make this code less nested with branches but
            -- I have kept it like this for simplicity rather than get fancy
            loaded <- loadGame gameRepository gameId
            case loaded of
                Left err ->
                    return $ Left err
                Right entity -> do
                    freshState <- mapGameState entity
                    -- It's important we recheck in the same transaction as writing the entry
                    -- to the cache that another thread hasn't inserted it in the meantime so
                    -- that we can make sure we're sharing the same TVars and channels, etc
                    atomically $ do
                        cached' <- M.lookup gameId gameMap
                        case cached' of
                            Just gameState -> Right <$> registerSharer gameState
                            Nothing -> do
                                M.insert freshState gameId gameMap
                                Right <$> registerSharer freshState
  where
    registerSharer :: GameState -> STM GameState
    registerSharer gameState = do
        modifyTVar' (gameConnections gameState) (+ 1)
        return gameState

    mapGameState :: GameEntity -> IO GameState
    mapGameState GameEntity { gameEntityBoard = b, gameEntityPlayer1 = p1, gameEntityPlayer2 = p2 } = do
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

In our `loadCachedEntry` function, in an STM transaction (via the [atomically](https://hackage.haskell.org/package/stm-2.5.3.1/docs/Control-Monad-STM.html#v:atomically) function) we check if the item is already in the cache (via `getCachedEntry`) and if it is not we insert it and return the newly inserted game entry.

This means that if there is a conflict in the middle of the transaction because another websocket thread has updated our cache in another concurrent transaction, the transaction will be retried and both threads will end up with the same game entry and see the same updates sent via the shared broadcast channel, etc.


## Taming the Herd

There's an improvement we can make to make sure that multiple threads don't try to load our game at the same time, wasting trips to the database just to throw the result away. This could happen in a thundering herd of observers to watch the latest tournament game between Noughts and Crosses grandmasters!

We will use an [MVar](https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Control-Concurrent-MVar.html) to allow a thread to lay claim to responsibility for loading the resource and signal any other threads waiting when it has done so. MVar's [readMVar](https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Control-Concurrent-MVar.html#v:readMVar) function allows the caller to wait (block) until it has been populated with a value (or it will return immediately if it's already populated with a value.)

The game entry in our game map will now either be a game loaded into the cache (`LoadedGame`) or a game that is currently being loaded by another websocket (`LoadingGame`.) `LoadingGame` will contain an MVar that can be read to receive a signal that the other thread has completed loading the game.

```
data GameEntry = LoadedGame GameState | LoadingGame (MVar ())
type GameMap = M.Map GameId GameEntry
```

When we first try to retrieve a game, we have to decide the action we need to take based on the cache state, which we represent with this type:

```haskell
data CacheDecision
    -- The game is already in the cache
    = Hit GameState
    -- We're awaiting another thread loading the game
    | Await (MVar ())
    -- We have claimed ownership over loading the game
    | Claimed (MVar ())
```

When we load a game we transactionally lay claim to loading the resource if it's not already loaded or is currently being loaded by writing a 'LoadingGame' entry to cache:

```haskell
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

This is performed in an STM transaction, so if multiple threads try to insert their 'MVar' claiming ownership then the losers of the race will retry the transaction and see the winning thread's claim when they read the map again. They will then block on it being populated by the winning thread using using [readMVar](https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Control-Concurrent-MVar.html#v:readMVar), which serves as a signal that they can now load the game from the cache.

When loading the resource we must make sure that threads don't wait on the MVar indefinetly if an error is thrown while retrieving the game so we clean up the claim if an error occurs while trying to load the game.

```haskell
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

When we think of garaunteeing the safe release of scarce resources, one of the options our mind turns to is the [resourcet](https://hackage.haskell.org/package/resourcet) package. This package allows us to use the type system to make sure a resource will be cleaned up when it is no longer being used. See [this blog entry](https://www.yesodweb.com/blog/2011/12/resourcet) for more on how it works if you are unfamilar with it.

We will use the resourcet package's [allocate](https://hackage.haskell.org/package/resourcet-1.3.0/docs/Control-Monad-Trans-Resource.html#v:allocate) function to register the action that must be taken when the game resource is no longer in scope (due to the websocket disconnecting or an exception being thrown.)

We have the 'gameConnections' variable already tracking the number of websocket handlers that are subscribed to the game. On connection, we will increment it via our `registerSharer` function as usual and on disconnection if the connection count has reached 0 we will remove it from the cache:

```haskell
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

We registered the `releaseGameState` function as the function that should be ran when the resource goes out of scope. It will check if there's any sharers left, and if there are not it will remove the game from the game map.

Again, the STM abstraction has served us well. The transaction inside `atomically` will be restarted if either the 'gameConnections' TVar or game map is updated by another websocket thread.

If a new websocket connection retrieves the game from the cache at the same time as it is being removed in another transaction, either:
* The transaction removing the cache entry is restarted, meaning the websocket doesn't end up with an orphaned game state and channel it will never get updates on
* Or the transaction retrieving the item from the cache is aborted, meaning the websocket thread has to insert it threshly and it will be visible to other websockets

*Note*: the conditional deletion can be achieved slightly more efficiently with stm-container's [focus](https://hackage-content.haskell.org/package/stm-containers-1.2.2/docs/StmContainers-Map.html#v:focus) function to avoid having to traverse the map a second time to do the deletion.

We can now use this game 'resource' in our websocket handler inside `runResourceT`:

```haskell
websocketHandler :: App -> GameId -> WebSocketsT Handler ()
websocketHandler app gameId = do
    userId <- lift requireAuthId
    websocketConnection <- ask
    runResourceT $ do
        result <- getGameState (gameRepository app) (gameMap app) gameId
        case result of
            Left _err -> return ()
            Right gameState ->
                liftIO $ race_
                    (handleIncomingMessages websocketConnection gameState userId)
                    (handleOutgoingMessages websocketConnection gameState userId)
```

We use [race_](https://hackage-content.haskell.org/package/async-2.2.6/docs/Control-Concurrent-Async.html#v:race_) to spawn two green threads to manage the Websocket connection.

`handleOutgoingMessages` will read from the shared broadcast channel and write to the websocket to inform the client of board changes.

`handleIncomingMessages` will read from the websocket to make moves on behalf of the user. It will validate the move, update the state and write to the shared broadcast channel.

When either `handleIncomingMessages` or `handleOutgoingMessages` fails on reading or writing to the socket due to it being disconnected, `runResourceT` will catch the error and run the `releaseGameState` function that we registered to run on resource deallocation in our `getGameState` function before rethrowing the error.

If you want to see a full implementation (mostly vibe coded other than what we've discussed here) of our demo noughts and crosses server, I've uploaded it to github [here](https://github.com/Happy0/vibe-noughts-and-crosses-demo). The [NoughtsAndCrosses.hs](https://github.com/Happy0/vibe-noughts-and-crosses-demo/blob/main/src/Handler/NoughtsAndCrosses.hs) file contains all the logic we've been discussing in this blog entry.

## Conclusion

This pattern can be applied in any situation where it's important that multiple threads need to share the same reference to something that is dynamically loaded. This is a requirement beyond that of a traditional cache where it doesn't particularly matter if an item is evicted from the cache (per some eviction policy) while there's still threads holding on to it, for example.

If you think you might find this pattern useful for something, I've published it as a library named [shared-resource-cache](https://hackage.haskell.org/package/shared-resource-cache). Source [here](https://github.com/Happy0/shared-resource-cache). 

*Note*: it works slightly differently to what we've explored here: when there are no 'sharers' of the given resource, it is not cleared from the cache immediately. Instead, it is cleared when there has been no sharers for at least a configured amount of time.

Feel free to send me a [game invite](https://wordify.gordo.life/create-lobby) on wordify. My username is happy0 :).