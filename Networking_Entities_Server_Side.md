**Note:** These guides assume familiarity with the [Lidgren networking library](http://code.google.com/p/lidgren-network-gen3/) and basic [game networking concepts](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking).

All networked types have both a client and server representation. The server type implements the `IServerEntity` interface, and registers a send table and, optionally, a lag compensation table with the `EntityServer`. Types must be registered in order from least to most derived and in the same order on client and server.

## Networking Server Types ##
`IServerEntity` consists of the `bool ShouldSend(NetPlayer recipient)` method. This method tells `EntityServer` whether `recipient` should receive updates about this entity. For example, we could save bandwidth by only sending players updates on entities visible to them.

Individual variables are networked by wrapping them in a `NetVariable` and adding a `SendInfo` entry to the send table. `NetVariable` keeps track of when the variable was last modified so we only send changes since the client's last acknowledged update. `SendInfo` takes:

  * An `ISerializer` to serialize the variable for transport. Serializers can contain state, such as the maximum value of the field, used to minimize bandwidth usage.
  * The name of the networked field and the `Type` it is a member of.
  * The `ChangeFrequency` of the field. Used for bandwidth optimization purposes. Unchanging fields will not incur overhead after the first time they are sent.
  * The array index if this variable is inside an array. EntNet supports fixed size arrays of `NetVariables` (ex: `NetVariable<int>[10]`). The field name should be the name of the array, and the parent type the `Type` containing the array.

Base class networked variables are included by appending the base class' send table to the derived class' table. Individual variables can then be excluded using the static `SendInfo.ExcludeSendInfo` method.

The order of `SendInfos` in the table does not matter except that it must match the order of the client side receive table.

```
public class Soldier_S : GameObject_S, IServerEntity
{
    static int MaxAmmo = 75;
    static List<SendInfo> sendTable;
    NetVariable<Vector2> position;
    NetVariable<int> ammo;
    NetVariable<int> ownerID;
    //...

    static void Initialize()
    {
        sendTable = new List<SendInfo>();
        sendTable.Add(new SendInfo(new Vec2Serializer(), "position", typeof(Soldier_S), ChangeFrequency.High));
        sendTable.Add(new SendInfo(new IntSerializer(MaxAmmo), "ammo", typeof(Soldier_S), ChangeFrequency.Medium));
        sendTable.Add(new SendInfo(new IntSerializer(), "ownerID", typeof(Soldier_S), ChangeFrequency.Never));
        sendTable.AddRange(GameObject_S.sendTable);
        SendInfo.ExcludeSendInfo(sendTable, "objectID", typeof(GameObject_S));

        //...

        Server.EntityServer.AddType(typeof(Soldier_S), sendTable, lagCompensationTable);
    }

    //...
}
```

## Lag Compensation ##
If this entity should be wound back during lag compensation we create a lag compensation table with a `LagCompensationInfo` entry for each field that should be wound back. Lag compensated fields do not have to be `NetVariables`. `LagCompensationInfo` takes:

  * The name of the lag compensated field and the `Type` it is a member of.
  * A bool indicating whether the field is a NetVariable.
  * An IInterpolator if the value should be interpolated between snapshot values. Interpolators can contain state, such as the maximum difference between two snapshot values beyond which we snap to the later value.
  * The array index if this field is inside an array. The field name should be the name of the array, and the parent type the `Type` containing the array.

Base class lag compensated variables are included by appending the base class lag compensation table to the derived class' table. Individual variables can then be excluded using the static `LagCompensationInfo.ExcludeLagCompInfo` method.

To use lag compensation wrap the lag compensated code between calls to `EntityServer.BeginLagCompensation(NetPlayer player, float currentTime)` and `Entityserver.EndLagCompensation()`.

```
public class Soldier_S : GameObject_S, IServerEntity
{
    //...
    static List<LagCompensationInfo> lagCompensationTable;
    Hitbox hitbox;
    Weapon weapon;
    NetPlayer owner;

    public static void Initialize()
    {
        //...

        lagCompensationTable = new List<LagCompensationInfo>();
        lagCompensationTable.Add(new LagCompensationInfo("hitbox", typeof(Soldier_S), false, new HitboxInterpolator()));

        Server.EntityServer.AddType(typeof(Soldier_S), sendTable, lagCompensationTable);
    }

    //...

    public void FireWeapon(float currentTime)
    {
        Server.EntityServer.BeginLagCompensation(owner, currentTime);
        Weapon.Fire();
        Server.EntityServer.EndLagCompensation();
    }

    //...
}
```

## Adding Entities ##
For standard entities we simply call `EntityServer.AddEntity(IServerEntity entity)`. For entities whose **creation** is predicted by the client, we call `EntityServer.AddPredictedEntity(IServerEntity entity, NetPlayer predictingPlayer, ushort predictingUserCmdID, byte predictionLink)`. EntNet uses this extra information to match the server side entity with the entity created on the client. In particular, `predictionLink` is a unique number identifying the locations on client and server where this entity gets created.

```
//on Client
void FireWeapon(UserCmd userCmd)
{
    Client.EntityClient.AddPredictedEntity(new Rocket_C(), userCmd.ID, PredictionLinks.FireRocket);
}

//on Server
void FireWeapon(NetPlayer player, UserCmd userCmd)
{
    Server.EntityServer.AddPredictedEntity(new Rocket_S(), player, userCmd.ID, PredictionLinks.FireRocket);
}

public static class PredictionLinks
{
    static const byte FireRocket = 0;
}
```

Note: Predicting the creations of multiple objects on the same link in the same UserCmd has undefined behavior. All entities will be created on the server but `EntityClient` will not necessarily match them to the correct client entities.

Note: Entities whose creations are predicted on the client are updated in the same UserCmd. Be sure to do the same on the server to avoid prediction errors.

## Updating Entities ##
Most entities can be updated inside the game loop, with networked members accessed through the `Read` and `Modify` properties of their `NetVariable` wrapper. The exception is predicted entities. Predicted entities are only updated when we receive a UserCmd from their owner. In addition, after the UserCmd is processed we must update the `LastUserCmd` property on the player's `NetPlayer` object.

```
//in Server
void ProcessUserCmd(NetIncomingMessage msg)
{
    UserCmd userCmd = new UserCmd(msg);
    NetPlayer player = (NetPlayer)msg.SenderConnection.Tag;
    Soldier_S soldier = (Soldier_S)NetPlayer.Tag;
    soldier.Update(userCmd, currentTime);
    player.LastUserCmd = userCmd.ID;
}

//in Soldier_S
void Update(UserCmd userCmd, float currentTime)
{
    position.Modify += userCmd.Velocity * userCmd.TimeStep;
    if(userCmd.Fire) FireWeapon(currentTime);
    if(rocket != null) rocket.Update(userCmd, currentTime);
}
```

Now all that's left is to [network the client side entities](Networking_Entities_Client_Side.md).