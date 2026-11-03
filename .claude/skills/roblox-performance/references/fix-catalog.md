# Fix Catalog: Physics, Humanoids, Rendering, Assets, Load Time

Read once the profile names one of these categories. Parent: `../SKILL.md`.

## Physics and Humanoids

- Anchor what doesn't need simulation. Keep adaptive stepping unless you need fixed
  (fixed forces 240 Hz). Reduce constraints, joints and ragdoll self-collision.
- `CollisionFidelity`: `Box` for small or non-interactive meshes, `Box`/`Hull` for
  small to medium, invisible primitive parts for large complex shapes. Geometry stays in
  memory even with collisions off, so also use a cheap fidelity on non-colliding meshes,
  with `CanCollide`, `CanTouch`, `CanQuery` all `false`. High `PhysicsParts` memory points here.
  744 makes fidelity recomputation async (**Pending**); don't assume geometry is ready
  immediately after changing it in tooling.
- `BasePart:SetNetworkOwner(nil)`, server-side: use for gameplay-critical parts clients
  shouldn't control, since client-owned physics can't be verified. Use conservatively: it
  may make physics feel jittery. Set it once; frequent changes cost (`filterStep`,
  `Distributed Physics Ownership`). Applies to the connected mechanism. Anchored parts
  are always server-owned, and anchoring loses a prior manual setting on a lone assembly.
- `BasePart:BindToCollisionSummaries` (announced 2026-10-06; in the reference; one early
  user hit a "not enabled" error, so verify in your place): per-frame contact data,
  cost scales with subscribed parts, nothing replicates, runs on the network owner.
  Staff describe it as generally much faster than `Touched`; no benchmark published.
- **Humanoids:** disable unused states with `SetStateEnabled`; use `AnimationController`
  on static NPCs; pool frequently respawned NPCs; spawn only near players; play NPC
  animations on the client; update `Motor6D.Transform`, not `C0`/`C1`; attach parts
  outside the avatar model hierarchy. `updateInvalidatedFastClusters` over 4 ms means
  avatar instantiation or modification is invalidating too much.
- Temporary objects: `Debris:AddItem` is current and not deprecated. Its hard cap is
  1,000 items, and the oldest is destroyed early past that, so `lifetime` is a maximum.
  Pool high-frequency effects like bullets and damage numbers. (pooling is judgment
  outside NPCs)

## Rendering and Assets

- **Instancing:** same `MeshContent` and identical `SurfaceAppearance` (or texture, or
  material) collapses into one draw call. Upload each mesh **once** and duplicate it in
  Studio. Importing a whole map creates a separate asset per copy, because the importer
  doesn't deduplicate. Decals, textures and particles batch poorly.
- Limit shadow casters (`CastShadow`, `Light.Shadows`, light range). Shadows disappear
  below quality level 4. Keep transparency at 0 or 1 where possible to avoid overdraw.
- `RenderFidelity` `Automatic` or `Performance`, not `Precise` everywhere. Skinned meshes
  moving without a Humanoid rebuild FastClusters; an embedded Humanoid forces one cluster,
  at a memory cost, so profile it.
- Culling is automatic for frustum and occlusion of parts, meshes, terrain. Lights added
  in 690; beams and trails 737 (**Pending**). Don't hand-roll culling first.
- **Texture memory is pixels:** 1024x1024 costs four times 512x512. Compressing or
  stripping alpha before upload saves disk, not memory. Usually 512 max; minor images under
  256. Prefer built-in materials, trim sheets, and `SurfaceAppearance.Color` tints.
- Load only the audio the current part of the game needs (`Sounds` category).
- **Raycast filter lists:** `ExcludeInstances` is up to 10x faster than
  `FilterDescendantsInstances` at 10,000 instances (714 **Pending**).

## Load Time

Keep `ReplicatedFirst` to the loading screen. Keep `ReplicatedStorage` lean, since
everything there replicates to every client before play; move large assets to
`ServerStorage`. `PreloadAsync` only the loading screen, key menu art and the spawn area,
never all of Workspace, and don't wait on `RequestQueueSize`. Offer **Skip Loading** if
it's long. An unexpectedly heavy join can mean a free model's scripts generating
thousands of instances: inspect them. Enable streaming.

## Last Resort

If server heartbeat stays low **after** optimization, Extended Services for Compute
reserves more CPU. It's paid and billed while enabled, and most games see no benefit.
That's the owner's decision, never yours.

## Sources

- https://create.roblox.com/docs/performance-optimization/improve, /physics/network-ownership, /cloud-services/extended-services, /reference/engine/classes/Debris, /reference/engine/classes/BasePart
- DevForum Collision Summaries: https://devforum.roblox.com/t/4918568

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
