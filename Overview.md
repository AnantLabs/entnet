As the name implies EntNet is all about entities. The library provides automatic entity replication and interpolation, input prediction, and lag compensation. This guide will provides a brief overview of each and when you would want to use them. For a more in depth guide see [this article](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking).

## Replication ##
Networked entities are registered with the `EntityServer` which replicates them to connected `EntityClients`. The `EntityServer` sends periodic updates to each `EntityClient` containing changes made since the `EntityClient`'s last acknowledged update. We reduce network load by sending these delta snapshots instead of the full state of every entity.

## Interpolation ##
Suppose we configure our client to receive 20 updates per second. If our networked entities were only rendered at the positions received from the server, moving entities would appear jittery. We solve this problem by interpolating smoothly between updates rather than snapping to the most recent one. Our interpolation period, the amount we go back in time for rendering, is fixed at twice the update period, 100ms in our example. This way even if one packet is lost we still have two updates to interpolate between. `EntityClient` automatically handles buffering and interpolating the updates, we just have to provide an `Interpolator` for each interpolated variable. The catch is that since we are rendering in the past we have to lead our targets when aiming in order to hit them. We solve this problem with lag compensation.

## Input Prediction ##
Suppose our client has a latency of 200ms. The client presses the FORWARD key and sends the command to the server. The server receives and processes the command, and sends the result back to the client. 200ms after the client presses the FORWARD key his avatar moves. This delay between user input and the corresponding result makes it hard to move and aim accurately. We solve this problem with input prediction. Rather than wait for the server's response, the client predicts the result of his input and moves instantly. He does this by executing the same movement code that the server executes. Usually this will result in an accurate prediction, but if there is a prediction error the server is authoritative and the client snaps to the proper state. `EntityClient` automatically predicts the results of input, buffers it, and runs prediction checking and correction. We just need to provide a `Comparer` to decide when the predicted and actual values are different enough to raise a prediction error, and possibly a `Cloner` to help `EntityClient` make deep copies of variables when it buffers them.   Input prediction can extend to any entity the local player controls including weapons.

## Lag Compensation ##
Suppose our client, again with 200ms latency, aims directly at his target, presses the FIRE key, and sends the command to the server. The server executes the command and the determines that the client has missed! Here's what happened:

T = 0:     The server sends the target position to the client.

T = 100ms: The update arrives on the client.

T = 200ms: The client renders 100ms in the past for interpolation, so it now renders the position in the update. The client aims, fires, and sends the command to the server.

T = 300ms: The server receives and executes the command. The target has changed position in the past 300ms, so the shot misses.

The client aims and fires based on the target's position at T = 0, but the server runs collision detection based on the target's position at T = 300. In order for the client to aim and shoot normally the server needs to roll the target's hitbox back to what the client would have seen when he fired. This is lag compensation, and the server time to execute the command at is:

execute time = current server time - client's latency - client's interpolation period

`EntityServer` handles rolling back lag compensated varaibles automatically for us. See [Networking\_Entities\_Server\_Side](Networking_Entities_Server_Side.md) for details.