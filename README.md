# Godot Simple Portal System

![Demo screenshot](https://github.com/Donitzo/godot-simple-portal-system/blob/main/images/screenshot.png)

### Table of Contents

1. **Introduction**
   - [Description](#description)
   - [Theory](#theory)
2. **Getting Started**
   - [About the Code](#about-the-code)
   - [About the Portal Script](#about-the-portal-script)
   - [About Modelling Portals](#about-modelling-portals)
   - [Setup Instructions](#setup-instructions)
3. **Advanced Features**
   - [Advanced Usage](#advanced-usage)
   - [Crossing a Portal](#crossing-a-portal)
   - [Teleportation](#teleportation)
   - [Raycasting](#raycasting)
4. **Feedback and Contributions**
   - [Feedback & Bug Reports](#feedback--bug-reports)

## Description

A Simple Portal System for Godot 4 (and 3 with a little work). Portals hopefully need no introduction. Just think of the game Portal and you get the idea. Non-nested portals are deceptively simple to implement, and can be incredibly powerful as both a gameplay mechanic and as a convenience feature to move players around your level, or to provide countless other fun special effects.

This simple portal system is meant as an educational example on how you can create portals in Godot. Consider it a starting point, as the relevant portal code has been documented clearly. You'll most likely need to adapt the code to fit the requirements of your game.

![Demo animation](https://github.com/Donitzo/godot-simple-portal-system/blob/main/images/demo.gif)

## Theory

In essence, portals are an illusion created by placing a virtual "exit camera" behind the exit portal. This camera replicates the main player camera's relative position to the entrance portal. As a result, both the player and the exit camera view the entrance and exit portals as if they occupy the same screen space. The visuals seen by the exit camera are rendered onto a 'viewport' in Godot (a render target), which is then overlaid on the entrance portal through a screen-space shader. This gives the impression that what's in front of the exit portal is visible through the entrance portal. 

We can further adjust the near clip distance of the exit camera frustum to attempt to not render objects behind the exit portal. While this won't handle all cases, it provides a simple enough solution for most scenarios. Advanced techniques such as discarding pixels of meshes behind the portal using a custom shader are also possible, but their implementation is dependent on the game in question.

![Portal theory](https://github.com/Donitzo/godot-simple-portal-system/blob/main/images/portals.png)

## About the code

This repository contains a small demo project which shows the portals in action. You can move around using WASD or the arrow keys, and look around using the mouse. A raycaster will show you which crates you hit as you move the mouse cursor. Another sample scene uses a fixed camera and shows a bunch of teleporting boxes.

If you want to use the portals in your own project, you only need these two files:

`src/shaders/portal.gdshader`
`src/scripts/portal.gd`

For an example of a fancier Portal-like portal, you can set this shader in the portal instance:

`src/shaders/spiraling_portal.gdshader`

The shaders are very simple and just renders a screen-space texture and handles fade-out.

If you don't want to make your own teleporting script, you can use one of these two:

`src/scripts/simple_teleport.gd`
`src/scripts/advanced_teleport.gd`

See the [teleportation section](#teleportation) for more information.

### About the Portal script

The portal script handles the creation of a viewport and positioning of the virtual exit camera. It also has functions for transforming between frames of reference and raycasting.

In `_process` the exit camera position is updated according to the main camera. I.e. the exit camera position mirrors the main (player) camera from the perspective of the exit portal.

The `_process` function also handles adjusting the near clipping plane of the exit camera, to find a compromise between not rendering objects behind the portal, and not cutting off the portal itself. This is done by getting the world position of the four X, Y corners of the entrance portal bounding box relative to (and scaled by) the exit camera. These corners are then projected onto the exit camera forward vector to get the near clipping distance which contain the corners within the camera frustum. The reason the entrance portal bounding box is used rather than the exit portal bounding box is because the entrance and exit meshes does not need to match each other, and you are looking through the entrance mesh, not the exit mesh. The exit may not even have any mesh at all.

## About Modelling Portals

First you need to model some portal meshes, or just use a plane or a box.

- The portal model surface should face -Y in Blender and +Z in Godot.
- To make a portal face another way, rotate the model object, not the mesh.
- The raycasting works by treating the portal as a flat surface centered at Z=0 in Godot. Flat portals work best in raycasting.

![Mesh](https://github.com/Donitzo/godot-simple-portal-system/blob/main/images/mesh.png)

## Setup Instructions

> **Note**: Portals are resource-intensive to render and consume texture memory. Disable portals that are far away from the player by using `disable_viewport_distance`. Furthermore, if you have many portals, consider enabling `destroy_disabled_viewport` to conserve texture memory, as this will destroy the viewport of disabled portals. However, this may potentially cause a small hickup when the viewport is re-created midgame.

> **Note**: Ensure that the parent hierarchy of the portal is uniformly scaled. Non-uniform scaling in the parent hierarchy can introduce skewing effects, which may lead to unexpected or incorrect behavior of the portal. However, scaling the portal itself in a non-uniform way is okay since it is handled by the transformations.

1. Attach the `portal.gd` script to two `MeshInstance3D` nodes that represent your portal surfaces.
2. Establish a connection between the two portals: Assign one portal to the `exit_portal` property of the other portal. For a one-way portal, leave one portal disconnected.
3. Set your primary camera to the `main_camera` property. If left unset, the portal will default to the primary camera if one exists.
4. Set the `vertical_viewport_resolution` of the viewport rendering the portal (which covers the entire screen). Set to 0 to automatically use the real screen resolution.
5. Define the fading range for the portal using `fade_out_distance_max` and `fade_out_distance_min`. Fades to `fade_out_color`.
6. Define the `disable_viewport_distance` for portal rendering. Put the value slightly above `fade_out_distance_max` to ensure the portal fades out completely before disabling itself. If you have a lot of portals it may be beneficial to enable `destroy_disabled_viewport`, which will free up texture memory when the portal is disabled by destroying the viewport, at the cost of the overhead of re-creating the viewport again later.
7. Define the `exit_scale` to adjust the exit portal's view scale. Imagine, for instance, a large door leading to a small door.
8. Adjust the `exit_near_subtract` if objects behind the exit portal get cut off. At 0 the portal exit is roughly cut at Z=0.
9. Set `exit_environment` to assign a specific environment to a portal. This is important if, for instance, you want to prevent environmental effects from being applied twice.

## Advanced Usage

These functions aid in transitioning between the portal entrance and exit frames of reference:

- `real_to_exit_transform(real:Transform3D) -> Transform3D`
- `real_to_exit_position(real:Vector3) -> Vector3`
- `real_to_exit_direction(real:Vector3) -> Vector3`

These are useful when you manipulate portal-associated objects. For instance, these functions would allow you to position a cloned spotlight at the exit portal:

```gd
clone_spotlight.global_transform = portal.real_to_exit_transform(real_spotlight.global_transform)
```

This code can also be used to teleport an object to the exit portal. Alternatively use `real_to_exit_position` if you only want to change the global position of your object.

> **Note**: Portals currently do not nest (ie, you can't see through two portals at once). To nest portals you'd have to update the exit_camera position in-between draw calls, or figure out a way to change the camera view matrix in-between rendering viewports. That is beyond the scope of this simple system, but if you got some nice ideas how to implement these things in godot, please [open an issue](https://github.com/Donitzo/godot-simple-portal-system/issues).

### Crossing a Portal

If the player in your game has a physical shape, such as hands or a body, having these parts cross the portal surface before teleporting the player to the exit will cause them to disappear. There are several solutions to this issue:

* Create a dummy player at the exit portal and move it using `real_to_exit_transform`. This stand-in will replace the missing player as it moves through the portal.
* Instead of a single portal, create two pairs of one-way portals with a buffer zone between them. This allows the player to fully enter the buffer zone before being teleported to the exit. Keep in mind that this forces you to model the same buffer zone on both sides of the portal.
* With a single pair of portals, create a buffer zone by moving the mesh surface backward along the Z-axis (+Y in Blender). Note that since the exit camera's near clipping range calculations assume the mesh is at Z=0, you may need to adjust the near clipping range through `exit_near_subtract` or another method. Anecdotally, in my game, I prefer making the portals shaped like a hard-edged lens like `/‾‾‾‾\` rather than `______`. This provides a buffer zone for the player to move their hand in while being flat around the portal edges. This kind of portal has depth if you look at it from the side, but it doesn't need you to model a buffer zone. See the example in the [teleportation section](#teleportation).

### Teleportation

Two teleportation scripts called `simple_portal_teleport.gd` and `advanced_portal_teleport.gd` are provided with the sample scene. The simple script simply moves and orients any node as it overlaps a teleporter area, while the advanced script also handles the physics of [RigidBody3D](https://docs.godotengine.org/en/stable/classes/class_rigidbody3d.html) and [CharacterBody3D](https://docs.godotengine.org/en/stable/classes/class_characterbody3d.html), as well as placeholder clones. These scripts attaches to a [Area3D](https://docs.godotengine.org/en/stable/classes/class_area3d.html) node, which is parented to a portal node. Create a [CollisionShape3D](https://docs.godotengine.org/en/stable/tutorials/physics/collision_shapes_3d.html) as a child node of the Area3D node. This shape should be right behind the portal, acting as a trigger to teleport objects entering the portal.

To trigger the teleportation, you need to define another [Area3D](https://docs.godotengine.org/en/stable/classes/class_area3d.html) with a [CollisionShape3D](https://docs.godotengine.org/en/stable/classes/class_collisionshape3d.html) on the object you want to teleport. Then, give it a Metadata element called `teleportable_root` to which you add a [NodePath](https://docs.godotengine.org/en/stable/classes/class_nodepath.html) pointing towards the root of the composite object you want to teleport (See the `Player` in the sample).

`PortalTeleport Area3D`:
- [x] **Monitoring**
- [ ] **Monitorable**

`Teleportable Object Area3D`:
- [ ] **Monitoring**
- [x] **Monitorable**

The portals in the example scene have a buffer zone built into them. This allows the player's camera to turn around fully without clipping through the portal. If your player has a hand, you may also need to move the teleport trigger in front of the camera so that the player teleports before their hand clips through the portal.

![Mesh](https://github.com/Donitzo/godot-simple-portal-system/blob/main/images/portal_buffer_zone.png)

The advanced version of the teleport script also handles the physics of teleporting bodies. It checks whether the velocity of the body is moving towards or away from the portal, and only teleports the body if it moves towards the portal. The position, orientation, linear velocity and angular velocity is then transformed into the exit portal's frame of reference.

![Physics boxes](https://github.com/Donitzo/godot-simple-portal-system/blob/main/images/physics_boxes.gif)

Using the advanced portal script, a portal clone can be used as a placeholder of the other side of the portal. The portal clone is specified as a Metadata element called `portal_clone` to which you add a [NodePath](https://docs.godotengine.org/en/stable/classes/class_nodepath.html) pointing to a visual placeholder (this is on the same [Area3D](https://docs.godotengine.org/en/stable/classes/class_area3d.html) with the `teleportable_root` field). The placeholder doesn't need to be fully functional as long as it looks like whatever the original object is. The clone node is hidden and shown automatically as the node passes portals.

Finally, if you want portals flush against surfaces (like in Portal), you may choose to disable wall collisions for the physics bodies as they approach the portal (how else could the node enter it). You can use the `collision_disable_area.gd` script to disable the collision masks of your choice for bodies which enter into the [Area3D](https://docs.godotengine.org/en/stable/classes/class_area3d.html) you set the script on. When the bodies enter the area, all layer masks set in the `disabled_collision_masks` Metadata field array of the body will be disabled until shortly after the body leaves the area again. See the `portal_demo.gd` scene for a practical example.

![Portal clone](https://github.com/Donitzo/godot-simple-portal-system/blob/main/images/portal_demo.gif)

The following is the hierarchy of a basic teleport example:

```
Player (Node3D, CharacterBody3D, or RigidBody3D)
└── TriggerArea (Area3D) # Trigger area for teleportation
    └── Metadata
        ├── teleportable_root: NodePath("..")
        └── portal_clone: NodePath("/Root/PlayerClone")

PortalEntrance (Portal, target portal = PortalExit)
└── Area3D (SimplePortalTeleport or AdvancedPortalTeleport)

PortalExit (Portal, target portal = PortalEntrance)
└── Area3D (SimplePortalTeleport or AdvancedPortalTeleport)

PlayerClone (MeshInstance3D or another node tree representing the player)
```

## Raycasting

Raycasting through portals can be complex. To simplify this, a built-in raycasting function is provided.

Define a function with this signature, in which you do your own raycasting against non-portal objects:

```gd
func _handle_raycast(from:Vector3, dir:Vector3, segment_distance:float, recursive_distance:float, recursions:int) -> bool:
```

Declare a callable for your function:

```gd
var callable:Callable = Callable(self, "_handle_raycast")
```

Then, use the built-in raycasting function as follows:

```gd
Portal.raycast(get_tree(), from_position, direction, callable, [max_distance=INF], [max_recursions=2], [ignore_backside=true])
```

`_handle_raycast` is always invoked at least once for the original ray. The `segment_distance` is `INF` if no portal was hit, or the distance to the hit portal. The function is invoked once more each time the ray recursively passes through another portal. A ray can be prematurely interrupted if `_handle_raycast` returns true, or if it hits the `max_recursions` limit. Return true if for example the current ray segment (within `segment_distance`) was blocked by something.

If you want to manually raycast, you can adapt the code in the Portal.raycast function to suit your requirements. Assuming you know a ray hits the entrance portal, you can get the corresponding exit ray using:

```gd
exit_position = portal.real_to_exit_position(position)
exit_direction = portal.real_to_exit_direction(direction)
```

## Feedback & Bug Reports

If you find any bugs or have feedback, please [open an issue](https://github.com/Donitzo/godot-simple-portal-system/issues) in the GitHub repository.
