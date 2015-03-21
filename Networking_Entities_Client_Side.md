**Note:** These guides assume familiarity with the [Lidgren networking library](http://code.google.com/p/lidgren-network-gen3/) and basic [game networking concepts](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking).

The client type implements the `IClientEntity` interface, and registers a receive table with the `EntityClient`. Again, types must be registered in order from least to most derived and in the same order on client and server.

## Networking Client Types ##
`IClientEntity` consists of:

  * `bool ShouldPredict()` Returns whether `EntityClient` should run prediction for this entity. Ex: If this weapon is carried by the local player.

  * `void Predict(IUserCmd iuserCmd, bool firstPrediction)` Executes the `UserCmd` on the entity. Our game's `UserCmd` class implements IUserCmd, and we cast back to `UserCmd` inside `Predict`. `firstPrediction` will be true for the first prediction, and false for repredictions during prediction correction. Check it to avoid replaying sound effects, repredicting entity creations, etc.

  * `EntityFreed()` Called when the `EntityClient` receives an update deleting this entity. Remove the entity from render-lists, release unmanaged resources, etc here.

Individual variables are networked by adding a `ReceiveInfo` to the receive table. `ReceiveInfo` takes:

  * An `IDeserializer` to reconstruct the variable. (Often the serializer and deserializer are the same class, implementing both interfaces. The interfaces are separated for flexibility.)
  * The name of the networked field and the `Type` it is a member of.
  * The `ChangeFrequency` of the field. Must be the same as in the corresponding `SendInfo`.
  * An `IInterpolator`, if this variable should be interpolated. Interpolators can contain state, such as the maximum difference between two snapshot values beyond which we snap to the later value instead of interpolate.
  * An `IComparer`, if this variable should be predicted. During prediction checking the comparer compares predicted and actual values and returns true if a prediction error should be raised. Comparers can contain state, such as the error tolerance of the comparison.
  * An `ICloner`, if this variable is predicted and is a reference type that requires a deep copy when being copied to the prediction buffer.
  * The array index if this variable is inside an array. The field name should be the name of the array, and the parent type the Type containing the array.

If a variable is both prediction and interpolation enabled interpolation only occurs if the entity is not currently predicting.

Base class networked variables are included by copying the base class' receive table contents to the derived class' table using the `SendInfo.Clone` method. Individual variables can then be excluded using the static `ReceiveInfo.ExcludeSendInfo` method.

The order of `ReceiveInfos` in the table does not matter except that it must match the order of the server side send table.

```
public class Soldier_C : GameObject_C, IClientEntity
{
    static int MaxAmmo = 75;
    static List<ReceiveInfo> receiveTable;
    Vector2 position;
    int ammo;
    long ownerID;
    //...

    static void Initialize()
    {
        receiveTable = new List<ReceiveInfo>();
        receiveTable.Add(new ReceiveInfo(new Vec2Serializer(), "position", typeof(Soldier_C), ChangeFrequency.High, new Vec2Interpolator(), new Vec2Comparer()));
        receiveTable.Add(new ReceiveInfo(new IntSerializer(MaxAmmo), "ammo", typeof(Soldier_C), ChangeFrequency.Medium, null, new IntComparer()));
        receiveTable.Add(new ReceiveInfo(new IntSerializer(), "ownerID", typeof(Solider_C), ChangeFrequency.Never));
        foreach(ReceiveInfo receiveInfo in GameObject_C.receiveTable) receiveTable.Add(receiveInfo.Clone());
        ReceiveInfo.ExcludeReceiveInfo(receiveTable, "objectID", typeof(GameObject_C));

        Client.EntityClient.AddType(typeof(Soldier_C), receiveTable);
    }

    //...

    bool ShouldPredict()
    {
        return ownerID == Client.PlayerID;
    }

    bool Predict(IUserCmd iUserCmd, bool firstPrediction)
    {
        UserCmd userCmd = (UserCmd)iUserCmd;
        position += userCmd.Velocity * userCmd.TimeStep;
        if(userCmd.Fire) FireWeapon();
    }

    bool EntityFreed()
    {
        Client.Soldiers.Remove(this);
    }
}
```

## Adding Entities ##
All networked types must have a parameterless constructor which `EntityClient` will invoke to create new instances of the entity.

To predict the **creation** of entities we call `EntityClient.AddPredictedEntity(IServerEntity entity, ushort predictingUserCmdID, byte predictionLink)`.

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

**These notes are worth repeating:**

Note: Predicting the creations of multiple objects on the same link in the same UserCmd has undefined behavior. All entities will be created on the server but `EntityClient` will not necessarily match them to the correct client entities.

Note: Entities whose creations are predicted on the client are updated in the same UserCmd. Be sure to do the same on the server to avoid prediction errors.

## Updating Entities ##
Predicted entities have `Predict` called automatically by the `EntityClient`. If there are client side only operations that need to be performed on entities such as rendering, updates to animation state, etc we generally have the entities place themselves on the relevant lists in their constructor. When the entities are deleted, `EntityFreed()` is called and the entities remove themselves from the lists.

```
public Soldier_C()
{
    Client.Soldiers.Add(this);
}

bool EntityFreed()
{
    Client.Soldiers.Remove(this);
}
```

And that's it! Check out the NetDemo project in the [Downloads](http://code.google.com/p/entnet/downloads/list) tab for a demonstration of most of EntNet's features. Please post questions, feedback, suggestions, and bug reports in the [forum](https://groups.google.com/forum/?fromgroups#!forum/entnet).