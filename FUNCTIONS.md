# Function library and pipelines

Every transformation link in `knowledge-graph.json` is written as calls to a small set of functions, in the spirit of basic Grasshopper components. Each built case can then be read as a short program: 2D prototype → transformations → In-Between States → Materialize → built case.

- The same function name means the same operation; the same call text (for example `PolarArray(Z, n=4)`) means the same parameters too.
- `→` chains steps on one link. `on` names the circles a step starts from. `+=` means a circle has a second incoming link.
- `Intent(goal) ≈ f(…)` records an architectural purpose that no single function captures, with the nearest function.
- The page's **Copy pipelines for an LLM** button produces this same text for the graph as it currently stands.

```text
Axes: X = span, Y = depth, Z = up (as in Rhino). 2D prototypes are drawn in the XZ plane, so a turn inside the drawing is a rotation about Y. 'tip' is the single corner opposite the two supports or loads; 'top', 'base', 'legs', 'ring', 'twigs' name the other parts.

## Functions
F1 Divide(member, n): Split a member into n equal segments by adding n − 1 nodes on it.
   1. Pick the member a–b.
   2. Add the nodes p_i = a + (i/n)(b − a) for i = 1 … n − 1.
   3. Replace a–b by the n short members a–p_1, …, p_(n−1)–b.
   Grasshopper: Divide Curve. Equilibrium: Kept: the new nodes are straight-through joints. Putting loads on them is part of the next step.
F2 Move(nodes, vector): Translate the chosen nodes; the members attached to them follow.
   1. Pick the nodes S.
   2. Move each node p in S to p + v.
   3. Members keep their end nodes, so they stretch and turn.
   Grasshopper: Move. Equilibrium: Re-check: member directions change, so the forces change, and a C member can turn into T.
F3 Truncate(node, t): Cut a corner: replace node v by two nodes on its two members and join them with a new member.
   1. Pick the corner v and its two members v–a and v–b.
   2. Mark the points at fraction t from v: v1 = v + t(a − v), v2 = v + t(b − v). With t = 0.5 they are the midpoints of the two sides.
   3. Connect v1–v2 with a new member.
   4. Give the load or support at v to v1 and v2, half each.
   5. Remove v and the short members v–v1, v–v2.
   Grasshopper: Evaluate Curve (t) + Line + Cull. Equilibrium: Re-check: the new member takes the force that turned the corner. It is C where the sides push (arch, V-strut) and T where they pull (cable). The trapezoid is balanced for the two equal loads that replace the single one.
F4 Branch(node, k, t): Gather the members of a fan into k groups; each group meets at a new branch node that is joined back to the fan node by one member.
   1. Pick the fan node v and its members.
   2. Split the members into k groups (k = 2: left | right halves; k = 3: symmetric pairs).
   3. For each group add a branch node at fraction t along the group's mean direction from v.
   4. Re-attach the group's members from v to its branch node.
   5. Connect v to each branch node.
   Grasshopper: Move + Line. Equilibrium: Re-check: each branch node must balance its group. A branch is straight under load when it points along the resultant of its group's loads.
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
F9 Rotate(axis, θ, loads): Turn the diagram, or the part named first, about an axis by θ.
   1. Pick what turns (default: everything), the axis and the angle θ.
   2. Rotate the nodes about the axis.
   3. Loads turn with the shape by default; write 'gravity' when they stay vertical.
   Grasshopper: Rotate / Rotate Axis / Rotate 3D. Equilibrium: Kept when the loads turn too (a rigid motion): every force is unchanged. With 'gravity' the loads keep pointing down, so re-check — a bar can switch from C to T.
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
F13 PolarArray(axis, n): Make n copies turned by 360°/n about an axis (Z = vertical).
   1. Pick the axis and the count n.
   2. Copy the diagram n − 1 times, each turned a further 360°/n.
   3. Merge nodes on the axis and any that coincide.
   Grasshopper: Polar Array. Equilibrium: Each copy stays balanced in its own plane. At shared nodes (the axis, the rings) the forces add, so check them; splayed feet need a ring.
F14 Superpose(A, B, …): Overlay diagrams in one frame, merge the nodes that coincide and add their loads.
   1. Place each diagram as stated (stack along an axis, cross on an axis, or overlap).
   2. Merge coincident nodes; a member shared by two diagrams carries the sum of their forces.
   3. Add the loads at each merged node.
   Grasshopper: Merge + Combine & Clean. Equilibrium: If every part is balanced and they meet only at nodes, the sum is balanced (superposition). New contacts or merged members need a re-check.
F15 Intent(goal) ≈ f(…): A step driven by an architectural purpose that no single function captures. Record the goal in words, then the nearest function and its parameters.
   1. Write the goal: who or what the change is for.
   2. Write ≈ and the nearest function call, for example Move(one side, +X).
   3. Draw the result and re-check it.
   Grasshopper: —. Equilibrium: Always re-check. The nearest function tells you which check applies.
Materialize: from the last diagram to the built case (material, sections, joints). Not a geometric function.

## Serpentine Gallery Pavilion, 2002
P1 = Triangular arch   # 2D prototype
SP1 = Remove(tie) on P1   # Two-bar strut
SP2 = Rotate(bar, Y, θ; gravity) on SP1   # Rotated struts
SP3 = Array(X, n=5, d<w) → Superpose() on SP2   # Overlapping band
SP4 = PolarArray(Z, n=4) → Rotate(X, 90°) on SP3   # Folded lattice box
CASE = Materialize on SP4   # Serpentine Gallery Pavilion, 2002

## HSBC Main Building, 1985
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
HS1 = Extend(tip, −Z, to ground) on P2   # Mast and stays
HS2 = Array(X, n=2) → Divide(hangers, n=floors) on HS1   # Twin-mast hanger frame
HS3 = Array(Z, n=5) on HS2   # Five hung tiers
CASE = Materialize on HS3   # HSBC Main Building, 1985

## Chiasso shed (Robert Maillart), 1924
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
P3 = Rotate(Y, 180°) on P1   # Triangular V-strut
MA1 = Extend(tip, −Z, to ground) on P3   # Y-column
P4 = FlipLoad() on P3   # Triangular cable
P4 += Rotate(Y, 180°) on P2
MA2 = Move(one corner, −Z) on MA1   # Tilted Y-column
MA3 = Divide(top & cable, n=8) → Connect(posts) on P4   # Beam string
MA4 = Move(top nodes, +Z; parabola) on MA3   # Lens truss
MA5 = Superpose(MA4, MA2 ×2) on MA4, MA2   # Lens on Y-columns
CASE = Materialize on MA5   # Chiasso shed (Robert Maillart), 1924

## Khudi Bari, 2020
P1 = Triangular arch   # 2D prototype
KB1 = Rotate(Z, 90°) on P1   # Triangle turned 90°
KB4 = Rotate(Z, 90°) → Array(X, n=2) on P1   # Twin frames
KB2 = Superpose(P1, KB1; cross) on KB1   # Crossed triangles
KB5 = Rotate(frames, X, ±θ) → Connect(pairs) on KB4   # Linked leaning frames
KB6 = Move(top nodes) on KB5   # Subsystem 2
KB3 = Remove(cross ties) → Connect(ring) on KB2   # Subsystem 1
KB3 += Superpose(S1, S2) on KB6
KB7 = Rotate(X, θ) on KB3   # Subsystem 3
KB8 = Superpose(S1, S2, S3) on KB3, KB6, KB7   # Combined frame
CASE = Materialize on KB8   # Khudi Bari, 2020

## Stuttgart Airport terminal, 1991
P1 = Triangular arch   # 2D prototype
P3 = Rotate(Y, 180°) on P1   # Triangular V-strut
ST1 = Divide(top, n=6) → Connect(tip, top nodes) on P3   # Fan strut
ST2 = Branch(tip, k=3, pairs) on ST1   # Fan on a trunk
ST3 = Branch(tip, k=2, halves) on ST2   # Y-branch tree
ST4 = Remove(middle twigs) → Rotate(roof, Y, 15°; gravity) on ST3   # Tree under a sloped roof
CASE = Materialize on ST4   # Stuttgart Airport terminal, 1991

## Sendai Mediatheque, 2001
P1 = Triangular arch   # 2D prototype
P3 = Rotate(Y, 180°) on P1   # Triangular V-strut
T1 = Truncate(tip, t=0.5) on P1   # Arch-cable trapezoid
T2 = Truncate(tip, t=0.5) on P3   # Arch-cable inverted
SM1 = Superpose(T1, T2, T1; stack Z) on T1, T2   # 2D column
SM2 = PolarArray(Z, n=6) on SM1   # Hexagonal column
SM3 = Remove(cross ties) → Connect(ring) on SM2   # Ringed tube
CASE = Materialize on SM3   # Sendai Mediatheque, 2001

## Winterthur station square roof, 2013
P1 = Triangular arch   # 2D prototype
P3 = Rotate(Y, 180°) on P1   # Triangular V-strut
T2 = Truncate(tip, t=0.5) on P3   # Arch-cable inverted
WI1 = Scale(sx=2, sz=0.4) → Extend(base, −Z, to ground) on T2   # Flat canopy unit
WI2 = PolarArray(Z, n=4) on WI1   # Four-way canopy
WI3 = Intent(room to wait for the bus) ≈ Move(one side, +X) on WI2   # Stretched canopy
CASE = Materialize on WI3   # Winterthur station square roof, 2013

## Notes on the links
P1 → P2: FlipLoad(). Reverse the apex load: the sides change from compression to tension and the tie becomes a strut.
P3 → P4: FlipLoad(). Reverse the loads: the V's sides change from compression to tension and the top tie becomes a strut.
P1 → P3: Rotate(Y, 180°). Turn the triangle upside down in its plane, loads included: every force stays the same and the apex becomes the support at the tip.
P2 → P4: Rotate(Y, 180°). Turn the tent upside down in its plane, loads included: every force stays the same.
P1 → T1: Truncate(tip, t=0.5). Cut the apex at the midpoints of the two sides and join them. The apex load becomes two equal loads, and the new top member is in compression: triangle → arch-cable trapezoid.
P3 → T2: Truncate(tip, t=0.5). Cut the bottom tip at the midpoints of the two legs and join them. The single support becomes two, and the new bottom member is in compression.
P1 → SP1: Remove(tie). Drop the tie: the two bars stay in compression and the supports now take the horizontal thrust.
SP1 → SP2: Rotate(bar, Y, θ; gravity). Turn one bar about the apex by θ (about 30° and 60° in the sketch) while the load stays vertical. At the larger angle the turned bar changes to tension.
SP2 → SP3: Array(X, n=5, d<w) → Superpose(). Five struts in a row, spaced closer than their width so they overlap between two horizontal lines; overlapping members merge into one band.
SP3 → SP4: PolarArray(Z, n=4) → Rotate(X, 90°). Wrap the band round a square plan: four copies about the vertical axis make the walls, and one copy laid flat (90° about X) makes the roof.
P2 → HS1: Extend(tip, −Z, to ground). Drop a mast from the apex to the ground. It carries the apex reaction in compression; the base corners now hang loads.
HS1 → HS2: Array(X, n=2) → Divide(hangers, n=floors). Two mast units side by side, sharing the middle corner. Hangers drop from the roof truss and are divided at every floor.
HS2 → HS3: Array(Z, n=5). Stack five suspension tiers; each tier carries its own group of floors.
P3 → ST1: Divide(top, n=6) → Connect(tip, top nodes). Divide the top tie into six; join each load point to the tip with its own bar (the middle point carries no load in the sketch).
ST1 → ST2: Branch(tip, k=3, pairs). Gather the bars in three symmetric pairs. Each pair meets the vertical axis at its own height, so a trunk grows from the tip.
ST2 → ST3: Branch(tip, k=2, halves). Regroup the bars into a left and a right group of three. Each group meets at one branch node partway up: a Y-shaped tree.
ST3 → ST4: Remove(middle twigs) → Rotate(roof, Y, 15°; gravity). Keep two twigs per branch, so four twigs carry R/4 each, and tilt the roof plane about 15° while the loads stay vertical.
P1 → KB1: Rotate(Z, 90°). Copy the triangle from plane N into plane M, turned 90° about the vertical axis.
KB1 → KB2: Superpose(P1, KB1; cross). Cross the two triangles on one vertical axis: four legs meet at one apex.
KB2 → KB3: Remove(cross ties) → Connect(ring). Take out the two ties that cross under the apex and tie the four feet with a ring instead.
P1 → KB4: Rotate(Z, 90°) → Array(X, n=2). Turn the triangle into plane M, then repeat it in a parallel plane M′.
KB4 → KB5: Rotate(frames, X, ±θ) → Connect(pairs). Lean the two frames apart (planes N and N′) and join corresponding nodes.
KB5 → KB6: Move(top nodes). Shift the top nodes to form Subsystem 2. To confirm: which nodes move, and where.
KB6 → KB3: Superpose(S1, S2). To confirm: KB8 already superposes subsystems 1, 2 and 3, so this link may not be needed.
KB3 → KB7: Rotate(X, θ). To confirm: Subsystem 3 read as Subsystem 1 tilted about X.
KB3 → KB8: Superpose(S1, S2, S3). Overlay the three subsystems in one frame; shared nodes add their forces.
KB6 → KB8: Superpose(S1, S2, S3). Overlay the three subsystems in one frame; shared nodes add their forces.
KB7 → KB8: Superpose(S1, S2, S3). Overlay the three subsystems in one frame; shared nodes add their forces.
T2 → WI1: Scale(sx=2, sz=0.4) → Extend(base, −Z, to ground). Stretch the inverted trapezoid flat and wide, then stand its two bottom nodes on posts to the ground.
WI1 → WI2: PolarArray(Z, n=4). Turn the canopy section four times about the vertical axis: a roof that cantilevers in four directions.
WI2 → WI3: Intent(room to wait for the bus) ≈ Move(one side, +X). Architectural intent: stretch one side out so people waiting for the bus stay under the roof. Nearest function: move the nodes on that side outward (a one-sided Scale about the posts); the short side is tied down.
T1 → SM1: Superpose(T1, T2, T1; stack Z). Stack T1, T2 and T1 along Z. Neighbours share a chord: struts at the narrow waists, ties at the wide levels.
T2 → SM1: Superpose(T1, T2, T1; stack Z). Stack T1, T2 and T1 along Z. Neighbours share a chord: struts at the narrow waists, ties at the wide levels.
SM1 → SM2: PolarArray(Z, n=6). Turn the 2D column six times, evenly, about the vertical axis: a hexagonal column with cross ties at each level.
SM2 → SM3: Remove(cross ties) → Connect(ring). Take out the ties that cross the column and tie each level with a hexagonal ring instead: a hollow tube.
P3 → MA1: Extend(tip, −Z, to ground). Extend the tip down to the ground as a column: the V becomes a Y.
MA1 → MA2: Move(one corner, −Z). Lower one top corner, so the tie slopes to meet the end of the lens truss.
P4 → MA3: Divide(top & cable, n=8) → Connect(posts). Divide the top strut into eight; the seven inner points carry the loads F. Give the cable a kink under each point and join them with posts: the cable takes the funicular (parabolic) shape of the loads.
MA3 → MA4: Move(top nodes, +Z; parabola). Lift the top-chord nodes on a parabola, so the top bows up while the cable bows down: a lens (fish-belly) truss.
MA4 → MA2: A constraint, not a transformation: the slope of MA2's tie matches the end of the lens truss MA4.
MA4 → MA5: Superpose(MA4, MA2 ×2). Set the lens truss on two tilted Y-columns; they share the end nodes.
MA2 → MA5: Superpose(MA4, MA2 ×2). Set the lens truss on two tilted Y-columns; they share the end nodes.
```
