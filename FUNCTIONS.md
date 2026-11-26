# Function library and pipelines

Every transformation link in `knowledge-graph.json` is written as calls to a small set of functions, in the spirit of basic Grasshopper components. Each built case can then be read as a short program: 2D prototype → geometry steps → intent steps → Materialize → built case.

- **Same template:** the same function with the same parameter names, e.g. `PolarArray(Z, n)`. Each case adds its own value (n = 4, n = 6), and "Values used so far" collects them into a range.
- `?` is a value still to fill; `≈` is a measured value. `[diagram to draw]` marks a step whose diagram is still blank.
- `→` chains steps on one link. `on` names the circles a step starts from. `+=` means a circle has a second incoming link. `[intent]` marks an architectural-intent step.
- The page's **Copy pipelines for an LLM** button produces this same text for the graph as it currently stands.

```text
Axes: X = span, Y = depth, Z = up (as in Rhino). 2D prototypes are drawn in the XZ plane, so a turn inside the drawing is a rotation about Y. 'tip' is the single corner opposite the two supports or loads; 'top', 'base', 'legs', 'ring', 'twigs' name the other parts.

Rules: Corner nodes of the 2D prototype are the starting points. A new point is made either between two existing nodes (a fraction t along a member, 0 to 1; t = 0.5 is the midpoint) or beyond a node (along a direction until a stop: ground, a length L, or a member). Whole-diagram moves (Rotate, Scale, Mirror) shift every node together. Name every numeric parameter (n=, t=, θ=, d=, sx=). Write ? for a value still to fill and ≈ for a measured value. Keep one or two functions per link. Every pipeline runs in three stages: geometry (exact rule values) → intent (Intent(type: goal) ≈ Function(values)) → Materialize.

## Functions
F1 Divide(member, n): Split a member into n equal segments by adding n − 1 nodes on it.
   1. Pick the member a–b.
   2. Add the nodes p_i = a + (i/n)(b − a) for i = 1 … n − 1.
   3. Replace a–b by the n short members a–p_1, …, p_(n−1)–b.
   Grasshopper: Divide Curve. Equilibrium: Kept: the new nodes are straight-through joints. Putting loads on them is part of the next step.
   Values used so far: n: ? ×2, 8
F2 Move(nodes, vector): Translate the chosen nodes; the members attached to them follow.
   1. Pick the nodes S.
   2. Move each node p in S to p + v.
   3. Members keep their end nodes, so they stretch and turn.
   Grasshopper: Move. Equilibrium: Re-check: member directions change, so the forces change, and a C member can turn into T.
   Values used so far: d: ? ×3 · f: ?
F3 Truncate(node, t): Cut a corner: replace node v by two nodes on its two members and join them with a new member.
   1. Pick the corner v and its two members v–a and v–b.
   2. Mark the points at fraction t from v: v1 = v + t(a − v), v2 = v + t(b − v). With t = 0.5 they are the midpoints of the two sides.
   3. Connect v1–v2 with a new member.
   4. Give the load or support at v to v1 and v2, half each.
   5. Remove v and the short members v–v1, v–v2.
   Grasshopper: Evaluate Curve (t) + Line + Cull. Equilibrium: Re-check: the new member takes the force that turned the corner. It is C where the sides push (arch, V-strut) and T where they pull (cable). The trapezoid is balanced for the two equal loads that replace the single one.
   Values used so far: t: 0.5 ×2
F4 Branch(node, k, t): Gather the members of a fan into k groups; each group meets at a new branch node that is joined back to the fan node by one member.
   1. Pick the fan node v and its members.
   2. Split the members into k groups (k = 2: left | right halves; k = 3: symmetric pairs).
   3. For each group add a branch node at fraction t along the group's mean direction from v.
   4. Re-attach the group's members from v to its branch node.
   5. Connect v to each branch node.
   Grasshopper: Move + Line. Equilibrium: Re-check: each branch node must balance its group. A branch is straight under load when it points along the resultant of its group's loads.
   Values used so far: k: 3, 2 · t: ? ×2
F5 Extend(node, direction, to): Add a new member from a node along a direction until it reaches a target (the ground or a length L); its far end becomes a support.
   1. Pick the node v and the direction d (for example −Z).
   2. Add the node e where the ray v + s·d meets the target.
   3. Connect v–e.
   4. The support at v moves to e (or e becomes a new support).
   Grasshopper: Line SDL. Equilibrium: Re-check: the new member carries what the support used to take at v — C for a mast or column, T for a hanger.
F6 Remove(members): Delete the chosen members (or nodes with their members).
   1. Pick the members (or nodes) to delete.
   2. Delete them; a node left with no members is deleted too.
   Grasshopper: Cull. Equilibrium: Re-check: the structure may turn into a mechanism or need new supports. Removing an arch's tie sends its thrust to the supports.
F7 Connect(a, b | ring): Add a member between two existing nodes, or a closed ring through a list of nodes.
   1. Pick the nodes a and b, or the ring a_1 … a_n.
   2. Add the member a–b (for a ring: a_1–a_2, …, a_n–a_1).
   Grasshopper: Line / Polyline (closed). Equilibrium: Re-check: adds a load path. A ring at splayed feet collects their outward push in T; a ring at a waist collects an inward pull in C.
F8 Scale(sx, sz, origin): Stretch the whole diagram about an origin, with different factors per axis if needed.
   1. Pick the origin o and the factors (sx, sy, sz).
   2. Move every node p to o + (sx(p − o)_x, sy(p − o)_y, sz(p − o)_z).
   Grasshopper: Scale NU. Equilibrium: Kept, with C and T unchanged (a parallel projection, Rankine). Force sizes change: in a 2D drawing vertical loads scale by sz/sx.
   Values used so far: sx: ? ×2 · sz: ? ×2
F9 Rotate(axis, θ, loads): Turn the diagram, or the part named first, about an axis by θ.
   1. Pick what turns (default: everything), the axis and the angle θ.
   2. Rotate the nodes about the axis.
   3. Loads turn with the shape by default; write 'gravity' when they stay vertical.
   Grasshopper: Rotate / Rotate Axis / Rotate 3D. Equilibrium: Kept when the loads turn too (a rigid motion): every force is unchanged. With 'gravity' the loads keep pointing down, so re-check — a bar can switch from C to T.
   Values used so far: θ: 180° ×2, ? ×3, 90° ×3, ±?
F10 Mirror(plane): Reflect the diagram in a plane.
   1. Pick the plane (vertical YZ, or horizontal XY).
   2. Reflect every node in it.
   3. Loads keep pointing down.
   Grasshopper: Mirror. Equilibrium: In a vertical plane: kept, forces unchanged. In a horizontal plane with loads still pointing down, every force changes sign, C↔T: Hooke's hanging chain turns an arch into a cable. Mirror(XY) = FlipLoad ∘ Rotate(Y, 180°) for a symmetric shape, so P1 → P4.
F11 FlipLoad(): Reverse every load, and so every reaction.
   1. Multiply every load vector by −1.
   2. Supports stay where they are; their reactions reverse too.
   Grasshopper: Negative (vector × −1). Equilibrium: Kept, and every member force changes sign: C becomes T and T becomes C.
F12 Array(direction, n, d): Make n copies spaced d along a direction; copies that touch share nodes.
   1. Pick the direction (X, Y or Z), the count n and the spacing d (default: the width, so copies just touch).
   2. Copy the diagram n − 1 times, each moved by d.
   3. Merge nodes that coincide.
   Grasshopper: Linear Array. Equilibrium: Each copy is balanced on its own. Where copies share nodes their forces add; when d is less than the width the copies overlap, so re-check.
   Values used so far: n: ?, 2
F13 PolarArray(axis, n): Make n copies turned by 360°/n about an axis (Z = vertical).
   1. Pick the axis and the count n.
   2. Copy the diagram n − 1 times, each turned a further 360°/n.
   3. Merge nodes on the axis and any that coincide.
   Grasshopper: Polar Array. Equilibrium: Each copy stays balanced in its own plane. At shared nodes (the axis, the rings) the forces add, so check them; splayed feet need a ring.
   Values used so far: n: 4 ×2, 6, ?
F14 Superpose(A, B, …): Overlay diagrams in one frame, merge the nodes that coincide and add their loads.
   1. Place each diagram as stated (stack along an axis, cross on an axis, or overlap).
   2. Merge coincident nodes; a member shared by two diagrams carries the sum of their forces.
   3. Add the loads at each merged node.
   Grasshopper: Merge + Combine & Clean. Equilibrium: If every part is balanced and they meet only at nodes, the sum is balanced (superposition). New contacts or merged members need a re-check.
   Values used so far: n: ?
F15 Intent(type: goal) ≈ f(…): A step driven by an architectural purpose that no single function captures. Record the type and the goal in words, then the nearest function and its values.
   1. Choose the type: use (people and activity), span (span, clearance, cantilever), light (light and view), site, or construction.
   2. Write the goal in a few words, e.g. Intent(use: room to wait for the bus).
   3. Write ≈ and the nearest function call with its values, e.g. ≈ Move(one side, +X, d=?).
   4. Give the intent its own link, at the step where the decision is made; draw the result and re-check it.
   Grasshopper: —. Equilibrium: Always re-check. The nearest function tells you which check applies.
Materialize: from the last diagram to the built case (material, sections, joints). Not a geometric function.

## Serpentine Gallery Pavilion, 2002
P1 = Triangular arch   # 2D prototype
SP1 = Remove(tie) on P1   # Two-bar strut
SP2 = Rotate(bar, Y, θ=?; gravity) on SP1   # Rotated struts
SP3 = Array(X, n=?, d<w) → Superpose() on SP2   # Overlapping band
SP4 = PolarArray(Z, n=4) on SP3   # Four walls [diagram to draw]
SP5 = Rotate(X, θ=90°) → Superpose() on SP4   # Folded lattice box
CASE = Materialize on SP5   # Serpentine Gallery Pavilion, 2002

## HSBC Main Building, 1985
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
HS1 = Extend(tip, −Z, to ground) on P2   # Umbrella
HS2 = Mirror(YZ, at corner) on HS1   # Twin umbrellas [diagram to draw]
HS3 = Divide(hangers, n=?) on HS2   # Hanger frame
HS4 = Scale(sx=?, sz=?) on HS3   # Scaled tier [diagram to draw]
HS5 = Superpose(tiers, n=?; stack Z) on HS4   # Stacked tiers
CASE = Materialize on HS5   # HSBC Main Building, 1985

## Chiasso shed (Robert Maillart), 1924
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
P3 = Rotate(Y, θ=180°) on P1   # Triangular V-strut
MA1 = Extend(tip, −Z, to ground) on P3   # Y-column
P4 = FlipLoad() on P3   # Triangular cable
P4 += Rotate(Y, θ=180°) on P2
MA2 = Move(one corner, −Z, d=?) on MA1   # Tilted Y-column
MA3 = Divide(top & cable, n=8) → Connect(posts) on P4   # Beam string
MA4 = Move(top nodes, +Z, f=?; parabola) on MA3   # Lens truss [diagram to draw]
MA5 = Superpose(MA4, MA2 ×2) on MA4, MA2   # Lens on Y-columns
CASE = Materialize on MA5   # Chiasso shed (Robert Maillart), 1924

## Khudi Bari, 2020
P1 = Triangular arch   # 2D prototype
KB1 = Rotate(Z, θ=90°) on P1   # Triangle turned 90°
KB4 = Rotate(Z, θ=90°) → Array(X, n=2) on P1   # Twin frames
KB2 = Superpose(P1, KB1; cross) on KB1   # Crossed triangles
KB5 = Rotate(frames, X, θ=±?) → Connect(pairs) on KB4   # Linked leaning frames
KB6 = Move(top nodes, d=?) on KB5   # Subsystem 2
KB3 = Remove(cross ties) → Connect(ring) on KB2   # Subsystem 1
KB3 += Superpose(S1, S2) on KB6
KB7 = Rotate(X, θ=?) on KB3   # Subsystem 3
KB8 = Superpose(S1, S2, S3) on KB3, KB6, KB7   # Combined frame
CASE = Materialize on KB8   # Khudi Bari, 2020

## Stuttgart Airport terminal, 1991
P1 = Triangular arch   # 2D prototype
P3 = Rotate(Y, θ=180°) on P1   # Triangular V-strut
ST1 = Divide(top, n=?) → Connect(tip, top nodes) on P3   # Fan strut
ST2 = Branch(tip, k=3, pairs, t=?) on ST1   # Fan on a trunk
ST3 = Branch(tip, k=2, halves, t=?) on ST2   # Y-branch tree
ST4 = Remove(middle twigs) on ST3   # Four twigs [diagram to draw]
ST5 = Rotate(roof, Y, θ=?; gravity) on ST4   # Tree under a sloped roof
ST6 = PolarArray(Z, n=?) on ST5   # Tree in 3D [diagram to draw]
CASE = Materialize on ST6   # Stuttgart Airport terminal, 1991

## Sendai Mediatheque, 2001
P1 = Triangular arch   # 2D prototype
P3 = Rotate(Y, θ=180°) on P1   # Triangular V-strut
T1 = Truncate(tip, t=0.5) on P1   # Arch-cable trapezoid
T2 = Truncate(tip, t=0.5) on P3   # Arch-cable inverted
SM1 = Superpose(T1, T2, T1; stack Z) on T1, T2   # 2D column
SM2 = PolarArray(Z, n=6) on SM1   # Hexagonal column
SM3 = Remove(cross ties) → Connect(ring) on SM2   # Ringed tube
CASE = Materialize on SM3   # Sendai Mediatheque, 2001

## Winterthur station square roof, 2013
P1 = Triangular arch   # 2D prototype
P3 = Rotate(Y, θ=180°) on P1   # Triangular V-strut
T2 = Truncate(tip, t=0.5) on P3   # Arch-cable inverted
WI1 = Scale(sx=?, sz=?) on T2   # Flat trapezoid [diagram to draw]
WI2 = Extend(base, −Z, to ground) on WI1   # Canopy on posts
WI3 = PolarArray(Z, n=4) on WI2   # Four-way canopy
WI4 = Intent(use: room to wait for the bus) ≈ Move(one side, +X, d=?) on WI3   # Stretched canopy [intent]
CASE = Materialize on WI4   # Winterthur station square roof, 2013

## Notes on the links
P1 → P2: FlipLoad(). Reverse the apex load: the sides change from compression to tension and the tie becomes a strut.
P3 → P4: FlipLoad(). Reverse the loads: the V's sides change from compression to tension and the top tie becomes a strut.
P1 → P3: Rotate(Y, θ=180°). Turn the triangle upside down in its plane, loads included: every force stays the same and the apex becomes the support at the tip.
P2 → P4: Rotate(Y, θ=180°). Turn the tent upside down in its plane, loads included: every force stays the same.
P1 → T1: Truncate(tip, t=0.5). Cut the apex at the midpoints of the two sides and join them. The apex load becomes two equal loads, and the new top member is in compression: triangle → arch-cable trapezoid.
P3 → T2: Truncate(tip, t=0.5). Cut the bottom tip at the midpoints of the two legs and join them. The single support becomes two, and the new bottom member is in compression.
P1 → SP1: Remove(tie). Drop the tie: the two bars stay in compression and the supports now take the horizontal thrust.
SP1 → SP2: Rotate(bar, Y, θ=?; gravity). Turn one bar about the apex while the load stays vertical. Fill θ (about 30° and 60° in your sketch). At the larger angle the turned bar changes to tension.
SP2 → SP3: Array(X, n=?, d<w) → Superpose(). Struts in a row, spaced closer than their width so they overlap between two lines (n = 5 in your sketch).
SP3 → SP4: PolarArray(Z, n=4). Four copies of the band about the vertical axis make the four walls of a square plan.
P2 → HS1: Extend(tip, −Z, to ground). Drop a mast from the apex to the ground. It carries the apex reaction in compression; the base corners now hang loads.
HS1 → HS2: Mirror(YZ, at corner). Copy the umbrella in a vertical plane through its outer corner, so two umbrellas share that corner.
HS3 → HS4: Scale(sx=?, sz=?). Set the proportion of one tier, its span and height.
P3 → ST1: Divide(top, n=?) → Connect(tip, top nodes). Divide the top tie and join each load point to the tip with its own bar (n = 6 in your sketch; the middle point carries no load).
ST1 → ST2: Branch(tip, k=3, pairs, t=?). Gather the bars in three symmetric pairs; each pair meets the vertical axis at its own height t, so a trunk grows from the tip.
ST2 → ST3: Branch(tip, k=2, halves, t=?). Regroup the bars into a left and a right group; each group meets at one branch node at height t (about 0.45 in your sketch).
ST3 → ST4: Remove(middle twigs). Keep two twigs per branch, so four twigs carry R/4 each.
P1 → KB1: Rotate(Z, θ=90°). Copy the triangle from plane N into plane M, turned 90° about the vertical axis.
KB1 → KB2: Superpose(P1, KB1; cross). Cross the two triangles on one vertical axis: four legs meet at one apex.
KB2 → KB3: Remove(cross ties) → Connect(ring). Take out the two ties that cross under the apex and tie the four feet with a ring instead.
P1 → KB4: Rotate(Z, θ=90°) → Array(X, n=2). Turn the triangle into plane M, then repeat it in a parallel plane M′.
KB4 → KB5: Rotate(frames, X, θ=±?) → Connect(pairs). Lean the two frames apart (planes N and N′) by ±θ and join corresponding nodes. Fill θ.
KB5 → KB6: Move(top nodes, d=?). Shift the top nodes to form Subsystem 2. Fill which nodes move and by how much.
KB6 → KB3: Superpose(S1, S2). To confirm: KB8 already superposes subsystems 1, 2 and 3, so this link may not be needed.
KB3 → KB7: Rotate(X, θ=?). Subsystem 3 read as Subsystem 1 tilted about X. Fill θ, or change the function if this reading is wrong.
KB3 → KB8: Superpose(S1, S2, S3). Overlay the three subsystems in one frame; shared nodes add their forces.
KB6 → KB8: Superpose(S1, S2, S3). Overlay the three subsystems in one frame; shared nodes add their forces.
KB7 → KB8: Superpose(S1, S2, S3). Overlay the three subsystems in one frame; shared nodes add their forces.
T2 → WI1: Scale(sx=?, sz=?). Stretch the inverted trapezoid flat and wide (about 2 and 0.4 in your sketch).
WI2 → WI3: PolarArray(Z, n=4). Turn the canopy section four times about the vertical axis: a roof that cantilevers in four directions.
WI3 → WI4: Intent(use: room to wait for the bus) ≈ Move(one side, +X, d=?). Architectural intent: stretch one side out so people waiting for the bus stay under the roof. Nearest function: move the nodes on that side outward (a one-sided Scale about the posts); the short side is tied down.
T1 → SM1: Superpose(T1, T2, T1; stack Z). Stack T1, T2 and T1 along Z. Neighbours share a chord: struts at the narrow waists, ties at the wide levels.
T2 → SM1: Superpose(T1, T2, T1; stack Z). Stack T1, T2 and T1 along Z. Neighbours share a chord: struts at the narrow waists, ties at the wide levels.
SM1 → SM2: PolarArray(Z, n=6). Turn the 2D column six times, evenly, about the vertical axis: a hexagonal column with cross ties at each level.
SM2 → SM3: Remove(cross ties) → Connect(ring). Take out the ties that cross the column and tie each level with a hexagonal ring instead: a hollow tube.
P3 → MA1: Extend(tip, −Z, to ground). Extend the tip down to the ground as a column: the V becomes a Y.
MA1 → MA2: Move(one corner, −Z, d=?). Lower one top corner, so the tie slopes to meet the end of the lens truss.
P4 → MA3: Divide(top & cable, n=8) → Connect(posts). Divide the top strut into eight; the seven inner points carry the loads F. Give the cable a kink under each point and join them with posts: the cable takes the funicular (parabolic) shape of the loads.
MA3 → MA4: Move(top nodes, +Z, f=?; parabola). Lift the top-chord nodes on a parabola of rise f, so the top bows up while the cable bows down: a lens (fish-belly) truss.
MA4 → MA2: A constraint, not a transformation: the slope of MA2's tie matches the end of the lens truss MA4.
MA4 → MA5: Superpose(MA4, MA2 ×2). Set the lens truss on two tilted Y-columns; they share the end nodes.
MA2 → MA5: Superpose(MA4, MA2 ×2). Set the lens truss on two tilted Y-columns; they share the end nodes.
SP4 → SP5: Rotate(X, θ=90°) → Superpose(). Lay one copy of the band flat as the roof and merge it with the walls: the lattice box.
HS2 → HS3: Divide(hangers, n=?). Hang the floors: divide each hanger at every floor of the tier (n = floors per tier).
HS4 → HS5: Superpose(tiers, n=?; stack Z). Stack the tiers on the masts (5 in your sketch).
ST4 → ST5: Rotate(roof, Y, θ=?; gravity). Tilt the roof plane while the loads stay vertical (about 15° in your sketch).
ST5 → ST6: PolarArray(Z, n=?). Turn the twigs about each branch to make the tree three-dimensional. Fill n.
WI1 → WI2: Extend(base, −Z, to ground). Stand the two bottom nodes on posts down to the ground.
```
