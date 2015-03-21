**Note:** These guides assume familiarity with the [Lidgren networking library](http://code.google.com/p/lidgren-network-gen3/) and basic [game networking concepts](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking).

As the name implies EntNet is all about entities. Networked entities are registered with the `EntityServer` which replicates them to connected `EntityClients`. The `EntityServer` then tracks and transmits state changes to each `EntityClient` based on their last acknowledged update.

## Server Setup ##
Server setup is fairly simple. We pass our `EntityServer` a Lidgren `NetServer` and a delegate that will create outgoing messages appended with an EntNet header.

```
public Server()
{
    NetPeerConfiguration config = new NetPeerConfiguration("Game");
    config.Port = 25000;
    netServer = new NetServer(config);
    netServer.Start();
    EntityServer = new EntityServer(netServer, CreateEntityMessage);
}

NetOutgoingMessage CreateEntityMessage()
{
    NetOutgoingMessage msg = netServer.CreateMessage();
    msg.Write((byte)GameMessageType.EntityMsg);
    return msg;
}
```

Players are represented on the server by the `NetPlayer` class. For convenience we attach each `NetPlayer` to the tag on its associated Lidgren `NetConnection` (this is not required by the library). The float in the `NetPlayer` constructor is the requested update period, fixed at 20 updates/sec for this game.

```
void RecieveMessages()
{
    NetIncomingMessage msg;
    while ((msg = netServer.ReadMessage()) != null)
    {
        switch (msg.MessageType)
        {
            case NetIncomingMessageType.Data:
                GameMessageType type = (GameMessageType)msg.ReadByte();
                switch (type)
                {
                    case GameMessageType.EntityMsg:
                        EntityServer.ReceiveMessage(msg, (NetPlayer)msg.SenderConnection.Tag);
                        break;
                    case GameMessageType.UserCmd:
                        ProcessUserCmd(msg);
                        break;
                }
                break;
            case NetIncomingMessageType.StatusChanged:
                NetConnectionStatus status = (NetConnectionStatus)msg.ReadByte();
                switch (status)
                {
                    case NetConnectionStatus.Disconnected:
                        RemovePlayer(msg);
                        break;
                    case NetConnectionStatus.Connected:
                        AddPlayer(msg);
                        break;
                }
                Console.WriteLine(status + " " + msg.ReadString());
                break;
            default:
                Console.WriteLine("Unhandled type: " + msg.MessageType);
                break;
        }
        netServer.Recycle(msg);
    }
}

void AddPlayer(NetIncomingMessage msg)
{
    NetPlayer player = new NetPlayer(msg.SenderConnection, 0.05f);
    EntityServer.Players.Add(player);
    msg.SenderConnection.Tag = player;
}

void RemovePlayer(NetIncomingMessage msg)
{
    NetPlayer player = (NetPlayer)msg.SenderConnection.Tag;
    EntityServer.Players.Remove(player);
}

void ProcessUserCmd(NetIncomingMessage msg)
{
    UserCmd userCmd = new UserCmd(msg);
    NetPlayer player = (NetPlayer)msg.SenderConnection.Tag;
    player.LastUserCmd = userCmd.ID;
}
```

Our server game loop might look like this:

```
public void Update(float currentTime, float timeStep)
{
    RecieveMessages();
    Simulate(currentTime, timeStep);
    EntityServer.Update(currentTime);
}
```

## Client Setup ##
Client setup is similar. The float in the `EntityClient` constructor is the interpolation time, which must be twice the requested update rate. This allows us to continue interpolating even with one dropped update.

```
public Client()
{
    NetPeerConfiguration config = new NetPeerConfiguration("Game");
    netClient = new NetClient(config);
    netClient.Start();
    netClient.Connect(new IPEndPoint(ip, 25000));
    EntityClient = new EntityClient(netClient, CreateEntityMessage, 0.1f);
}

NetOutgoingMessage CreateEntityMessage()
{
    NetOutgoingMessage msg = netClient.CreateMessage();
    msg.Write((byte)GameMessageType.EntityMsg);
    return msg;
}
```

Note that the `EntityClient` handles recycling its own messages.

```
void RecieveMessages()
{
    NetIncomingMessage msg;
    while ((msg = netClient.ReadMessage()) != null)
    {
        switch (msg.MessageType)
        {
            case NetIncomingMessageType.Data:
                GameMessageType type = (GameMessageType)msg.ReadByte();
                switch (type)
                {
                    case GameMessageType.EntityMsg:
                        EntityClient.ReceiveMessage(msg);
                        break;
                }
                break;
            default:
                Console.WriteLine("Unhandled type: " + msg.MessageType);
                netClient.Recycle(msg);
                break;
        }
    }
}
```

Our client game loop might look like this:

```
public void Update(float currentTime, float timeStep)
{
    ReceiveMessages();
    UserCmd userCmd = GetInput();
    EntityClient.Update(currentTime, userCmd);

    NetOutgoingMessage msg = netClient.CreateMessage();
    msg.Write((byte)GameMessageType.UserCmd);
    userCmd.Serialize(msg);
    netClient.SendMessage(msg, NetDeliveryMethod.UnreliableSequenced);
}
```

That's it! EntNet handles the rest. Now we just need some [entities to network](Networking_Entities_Server_Side.md).