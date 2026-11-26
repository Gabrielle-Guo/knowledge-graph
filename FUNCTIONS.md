# Function library and pipelines

Every transformation link in `knowledge-graph.json` is written as calls to a small set of functions, in the spirit of basic Grasshopper components. Each built case can then be read as a short program: 2D prototype → geometry steps → intent steps → Materialize → built case.

- **Same template:** the same function with the same parameter names, e.g. `PolarArray(Z, n)`. Each case adds its own value (n = 4, n = 6), and "Values used so far" collects them into a range.
- `?` is a value still to fill; `≈` is a measured value. `[diagram to draw]` marks a step whose diagram is still blank.
- `→` chains steps on one link. `on` names the circles a step starts from. `+=` means a circle has a second incoming link. `[intent]` marks an architectural-intent step, and `[stabilize]` a step that adds supports or ties so form and force close again.
- The page's **Copy pipelines for an LLM** button produces this same text for the graph as it currently stands.

```text
Axes: X = span, Y = depth, Z = up (as in Rhino). 2D prototypes are drawn in the XZ plane, so a turn inside the drawing is a rotation about Y. 'tip' is the single corner opposite the two supports or loads; 'top', 'base', 'legs', 'ring', 'twigs' name the other parts.

Rules: Corner nodes of the 2D prototype are the starting points. A new point is made either between two existing nodes (a fraction t along a member, 0 to 1; t = 0.5 is the midpoint) or beyond a node (along a direction until a stop: ground, a length L, or a member). Whole-diagram moves (Rotate, Scale, Mirror) shift every node together. Name every numeric parameter (n=, t=, θ=, d=, s=). Write ? for a value still to fill and ≈ for a measured value. Keep one or two functions per link. Every pipeline runs in stages: geometry (exact rule values) → intent (Intent(type: goal) ≈ Function(values)) → stabilize (after an intent step the form and force diagrams may no longer close, so supports or ties are added, often taken from an original type) → Materialize.

## Functions
F1 Divide(member, n): Split a member into n equal segments by adding n − 1 nodes on it.
   1. Pick the member a–b.
   2. Add the nodes p_i = a + (i/n)(b − a) for i = 1 … n − 1.
   3. Replace a–b by the n short members a–p_1, …, p_(n−1)–b.
   Grasshopper: Divide Curve. Equilibrium: Kept: the new nodes are straight-through joints. Putting loads on them is part of the next step.
   Values used so far: n: ? ×3, 8
F2 Move(nodes, vector): Translate the chosen nodes; the members attached to them follow.
   1. Pick the nodes S.
   2. Move each node p in S to p + v.
   3. Members keep their end nodes, so they stretch and turn.
   Grasshopper: Move. Equilibrium: Re-check: member directions change, so the forces change, and a C member can turn into T.
   Values used so far: d: ? ×5
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
   Values used so far: d: ? ×3
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
   Values used so far: sx: ? · sz: ? ×2 · s: ? ×3
F9 Rotate(axis, θ, loads): Turn the diagram, or the part named first, about an axis by θ.
   1. Pick what turns (default: everything), the axis and the angle θ.
   2. Rotate the nodes about the axis.
   3. Loads turn with the shape by default; write 'gravity' when they stay vertical.
   Grasshopper: Rotate / Rotate Axis / Rotate 3D. Equilibrium: Kept when the loads turn too (a rigid motion): every force is unchanged. With 'gravity' the loads keep pointing down, so re-check — a bar can switch from C to T.
   Values used so far: θ: 180° ×2, ? ×8, 90° ×3, ±? ×2
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
   Values used so far: n: ? ×5, 2 ×2, 7
F13 PolarArray(axis, n): Make n copies turned by 360°/n about an axis (Z = vertical).
   1. Pick the axis and the count n.
   2. Copy the diagram n − 1 times, each turned a further 360°/n.
   3. Merge nodes on the axis and any that coincide.
   Grasshopper: Polar Array. Equilibrium: Each copy stays balanced in its own plane. At shared nodes (the axis, the rings) the forces add, so check them; splayed feet need a ring.
   Values used so far: n: 6, ?, 4
F14 Superpose(A, B, …): Overlay diagrams in one frame, merge the nodes that coincide and add their loads.
   1. Place each diagram as stated (stack along an axis, cross on an axis, or overlap).
   2. Merge coincident nodes; a member shared by two diagrams carries the sum of their forces.
   3. Add the loads at each merged node.
   Grasshopper: Merge + Combine & Clean. Equilibrium: If every part is balanced and they meet only at nodes, the sum is balanced (superposition). New contacts or merged members need a re-check.
F15 Intent(type: goal) ≈ f(…): A step driven by an architectural purpose that no single function captures. Record the type and the goal in words, then the nearest function and its values.
   1. Choose the type: use (people and activity), span (span, clearance, cantilever), light (light and view), form (the shape it should read as, e.g. curved cables), site, or construction.
   2. Write the goal in a few words, e.g. Intent(use: room to wait for the bus).
   3. Write ≈ and the nearest function call with its values, e.g. ≈ Move(one side, +X, d=?).
   4. Give the intent its own link, at the step where the decision is made; draw the result and re-check it.
   Grasshopper: —. Equilibrium: Always re-check. An intent step usually breaks equilibrium, so it is followed by stabilize steps that add supports or ties.
Materialize: from the last diagram to the built case (material, sections, joints). Not a geometric function.

## HSBC Main Building, 1985
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
HS1 = Extend(tip, −Z, to ground) on P2   # Umbrella
HS2 = Array(X, n=2) on HS1   # Twin umbrellas
HS3 = Extend(corners, −Z, d=?) on HS2   # Hung floors
HS4 = Scale(s=?) on HS3   # Scaled tier
HS5 = Extend(masts, −Z, to ground) on HS4   # Masts to ground
HS6 = Array(Z, n=?) on HS5   # Stacked tiers
CASE = Materialize on HS6   # HSBC Main Building, 1985

## Millau Viaduct, 2004
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
HS1 = Extend(tip, −Z, to ground) on P2   # Umbrella
MI1 = Divide(base, n=?) → Connect(mast, base nodes) on HS1   # Cable fan
MI2 = Extend(mast, −Z, d=?) on MI1   # Tall pylon
MI3 = Array(X, n=7) on MI2   # Seven pylons [diagram to draw]
CASE = Materialize on MI3   # Millau Viaduct, 2004

## Khudi Bari, 2020
P1 = Triangular arch   # 2D prototype
KB1 = Rotate(Z, θ=90°) on P1   # Triangle turned 90°
KB4 = Rotate(Z, θ=90°) → Array(X, n=2) on P1   # Twin frames
KB2 = Superpose(P1, KB1; cross) on KB1   # Crossed triangles
KB5 = Rotate(frames, X, θ=±?) → Connect(pairs) on KB4   # Linked leaning frames
KB6 = Move(top nodes, d=?) on KB5   # Subsystem 2
KB3 = Remove(cross ties) → Connect(ring) on KB2   # Subsystem 1
KB3 += Superpose(S1, S2) on KB6
KB7 = Rotate(X, θ=?) on KB6   # Subsystem 3
KB8 = Superpose(S1, S2, S3) on KB3, KB6, KB7   # Combined frame
CASE = Materialize on KB8   # Khudi Bari, 2020

## Serpentine Gallery Pavilion, 2002
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
SP1 = Remove(tie) on P1   # Two-bar strut
IG1 = Extend(base, −Z, d=?) on P2   # Tent on legs
SP2 = Rotate(bar, Y, θ=?; gravity) on SP1   # Rotated struts
SP3 = Array(X, n=?, d<w) → Superpose() on SP2   # Overlapping band
SP4e = Rotate(Z, θ=90°) → Superpose(cross) on IG1   # Crossed frames
SP4a = Rotate(bars, Y, θ=?; gravity) on SP3   # Wall A
SP4b = Rotate(bars, Y, θ=?; gravity) on SP3   # Wall B
SP4c = Rotate(bars, Y, θ=?; gravity) on SP3   # Wall C
SP4d = Rotate(bars, Y, θ=?; gravity) on SP3   # Wall D
SP5 = Superpose(walls ×4, roof; box) on SP4b, SP4c, SP4d, SP4e, SP4a   # Folded lattice box
CASE = Materialize on SP5   # Serpentine Gallery Pavilion, 2002

## Stuttgart Airport terminal, 1991
P1 = Triangular arch   # 2D prototype
P3 = Rotate(Y, θ=180°) on P1   # Triangular V-strut
ST1 = Divide(top, n=?) → Connect(tip, top nodes) on P3   # Fan strut
ST3 = Branch(tip, k=2, halves, t=?) → Remove(middle twigs) on ST1   # Y-branch tree
ST5 = Rotate(roof, Y, θ=?; gravity) on ST3   # Tree under a sloped roof
ST6 = PolarArray(Z, n=?) on ST5   # Tree in 3D
CASE = Materialize on ST6   # Stuttgart Airport terminal, 1991

## David S. Ingalls Rink, 1958
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
T3a = Divide(sides, n=?) → Move(nodes, +Z; parabola) on P1   # Parabolic arch [diagram to draw]
IG1 = Extend(base, −Z, d=?) on P2   # Tent on legs
T3 = Connect(hangers) on T3a   # Arch-cable tied arch
IG2 = Array(Y, n=?) → Scale(s=?) on IG1   # Frames along the spine
IG3 = Mirror(XY, copy) → Superpose() on T3   # Lens of two arches
IG4 = Rotate(frames, X, θ=±?) on IG3   # Arches turned apart
IG5 = Superpose(spine frames, cable net) on IG2, IG4   # Spine and cable net
IG6 = Intent(form: curved cables) ≈ Move(cable nodes, d=?; curve) on IG5   # Adjusted roof [intent]
CASE = Materialize on IG6   # David S. Ingalls Rink, 1958

## Sendai Mediatheque, 2001
P1 = Triangular arch   # 2D prototype
P3 = Rotate(Y, θ=180°) on P1   # Triangular V-strut
T1 = Truncate(tip, t=0.5) on P1   # Arch-cable trapezoid
T2 = Truncate(tip, t=0.5) on P3   # Arch-cable inverted
SM1 = Superpose(T1, T2, T1; stack Z) on T1, T2   # 2D column
SM2 = PolarArray(Z, n=6) on SM1   # Hexagonal column
SM3 = Remove(cross ties) → Connect(ring) on SM2   # Ringed tube
CASE = Materialize on SM3   # Sendai Mediatheque, 2001

## Sofiero Footbridge, 2018
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
P3 = Rotate(Y, θ=180°) on P1   # Triangular V-strut
T3a = Divide(sides, n=?) → Move(nodes, +Z; parabola) on P1   # Parabolic arch [diagram to draw]
P4 = FlipLoad() on P3   # Triangular cable
P4 += Rotate(Y, θ=180°) on P2
T3 = Connect(hangers) on T3a   # Arch-cable tied arch
MA3 = Divide(top & cable, n=8) → Connect(posts) on P4   # Beam string
MA3 += Mirror(XY) on T3
SO1 = Array(X, n=?) → Scale(s=?) on P4   # Graded triangles
SO2 = Rotate(Y, θ=?; gravity) on MA3   # Sloped beam string
SO3 = Superpose(SO1, SO2) on SO2, SO1   # Truss on a slope
CASE = Materialize on SO3   # Sofiero Footbridge, 2018

## Winterthur station square roof, 2013
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
P3 = Rotate(Y, θ=180°) on P1   # Triangular V-strut
P4 = FlipLoad() on P3   # Triangular cable
P4 += Rotate(Y, θ=180°) on P2
T2 = Truncate(tip, t=0.5) on P3   # Arch-cable inverted
WI0 = Move(one corner, −X, d=?) → Extend(corner, −Z, to ground) on P3   # Back-stayed cantilever
WI1 = Scale(sx=?, sz=?) on T2   # Flat trapezoid
WI2 = Extend(base, −Z, to ground) on WI1   # Canopy on posts
WI3 = PolarArray(Z, n=4) on WI2   # Four-way canopy
WI4 = Intent(use: room to wait for the bus) ≈ Move(one side, +X, d=?) on WI3   # Stretched, tied down [intent]
WI4 += Superpose(canopy, tie-down) on WI0 [stabilize]
WI5 = Superpose(canopy, P4; tie-down) on WI4, P4   # Tie-down added [stabilize]
WI6 = Extend(post, −Z, to ground) → Connect(ties) on WI5   # Braced canopy [stabilize]
CASE = Materialize on WI6   # Winterthur station square roof, 2013

## Chiasso shed (Robert Maillart), 1924
P1 = Triangular arch   # 2D prototype
P2 = FlipLoad() on P1   # Triangular tent
P3 = Rotate(Y, θ=180°) on P1   # Triangular V-strut
T3a = Divide(sides, n=?) → Move(nodes, +Z; parabola) on P1   # Parabolic arch [diagram to draw]
MA1 = Extend(tip, −Z, to ground) on P3   # Y-column
P4 = FlipLoad() on P3   # Triangular cable
P4 += Rotate(Y, θ=180°) on P2
T3 = Connect(hangers) on T3a   # Arch-cable tied arch
MA2 = Move(one corner, −Z, d=?) on MA1   # Tilted Y-column
MA3 = Divide(top & cable, n=8) → Connect(posts) on P4   # Beam string
MA3 += Mirror(XY) on T3
MA4 = Scale(middle post, sz=?) on MA3   # Lens truss
MA5 = Superpose(MA4, MA2 ×2) on MA4, MA2   # Lens on Y-columns
MA6 = Array(Y, n=?) on MA5   # Bays in 3D
CASE = Materialize on MA6   # Chiasso shed (Robert Maillart), 1924

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
P2 → HS1: Extend(tip, −Z, to ground). Drop a mast from the apex to the ground. It carries the apex reaction in compression; the base corners now hang loads.
P3 → ST1: Divide(top, n=?) → Connect(tip, top nodes). Divide the top tie and join each load point to the tip with its own bar (n = 6 in your sketch; the middle point carries no load).
ST1 → ST2: Branch(tip, k=3, pairs, t=?). Gather the bars in three symmetric pairs; each pair meets the vertical axis at its own height t, so a trunk grows from the tip.
P1 → KB1: Rotate(Z, θ=90°). Copy the triangle from plane N into plane M, turned 90° about the vertical axis.
KB1 → KB2: Superpose(P1, KB1; cross). Cross the two triangles on one vertical axis: four legs meet at one apex.
KB2 → KB3: Remove(cross ties) → Connect(ring). Take out the two ties that cross under the apex and tie the four feet with a ring instead.
P1 → KB4: Rotate(Z, θ=90°) → Array(X, n=2). Turn the triangle into plane M, then repeat it in a parallel plane M′.
KB4 → KB5: Rotate(frames, X, θ=±?) → Connect(pairs). Lean the two frames apart (planes N and N′) by ±θ and join corresponding nodes. Fill θ.
KB5 → KB6: Move(top nodes, d=?). Shift the top nodes to form Subsystem 2. Fill which nodes move and by how much.
KB6 → KB3: Superpose(S1, S2). To confirm: KB8 already superposes subsystems 1, 2 and 3, so this link may not be needed.
KB3 → KB8: Superpose(S1, S2, S3). Overlay the three subsystems in one frame; shared nodes add their forces.
KB6 → KB8: Superpose(S1, S2, S3). Overlay the three subsystems in one frame; shared nodes add their forces.
T2 → WI1: Scale(sx=?, sz=?). Stretch the inverted trapezoid flat and wide (about 2 and 0.4 in your sketch).
T1 → SM1: Superpose(T1, T2, T1; stack Z). Stack T1, T2 and T1 along Z. Neighbours share a chord: struts at the narrow waists, ties at the wide levels.
T2 → SM1: Superpose(T1, T2, T1; stack Z). Stack T1, T2 and T1 along Z. Neighbours share a chord: struts at the narrow waists, ties at the wide levels.
SM1 → SM2: PolarArray(Z, n=6). Turn the 2D column six times, evenly, about the vertical axis: a hexagonal column with cross ties at each level.
SM2 → SM3: Remove(cross ties) → Connect(ring). Take out the ties that cross the column and tie each level with a hexagonal ring instead: a hollow tube.
P3 → MA1: Extend(tip, −Z, to ground). Extend the tip down to the ground as a column: the V becomes a Y.
MA1 → MA2: Move(one corner, −Z, d=?). Lower one top corner, so the tie slopes to meet the end of the lens truss.
P4 → MA3: Divide(top & cable, n=8) → Connect(posts). Divide the top strut into eight; the seven inner points carry the loads F. Give the cable a kink under each point and join them with posts: the cable takes the funicular (parabolic) shape of the loads.
MA3 → MA4: Scale(middle post, sz=?). Stretch the middle post: the top chord rises to a ridge and the middle post changes from compression to tension (blue → red). Not an intent step.
MA4 → MA2: A constraint, not a transformation: the slope of MA2's tie matches the end of the lens truss MA4.
MA4 → MA5: Superpose(MA4, MA2 ×2). Set the lens truss on two tilted Y-columns; they share the end nodes.
MA2 → MA5: Superpose(MA4, MA2 ×2). Set the lens truss on two tilted Y-columns; they share the end nodes.
ST5 → ST6: PolarArray(Z, n=?). Turn the twigs about each branch to make the tree three-dimensional. Fill n.
WI1 → WI2: Extend(base, −Z, to ground). Stand the two bottom nodes on posts down to the ground.
HS1 → HS2: Array(X, n=2). Two umbrellas side by side, sharing the corner between them.
HS2 → HS3: Extend(corners, −Z, d=?). Hangers drop from the corners; d is the hung height.
HS3 → HS4: Scale(s=?). Scale the hung tier (your arrow pointed the other way; it now runs HS3 → HS4).
HS4 → HS5: Extend(masts, −Z, to ground). Run the masts down to the ground.
HS5 → HS6: Array(Z, n=?). Stack the tiers on the masts (five in your earlier sketch).
MI1 → MI2: Extend(mast, −Z, d=?). Lengthen the pylon below the deck into a tall pier. Fill d.
HS1 → MI1: Divide(base, n=?) → Connect(mast, base nodes). From the umbrella HS1 (P2 with its mast to the ground): divide the deck and stay each deck point from the mast.
MI1 → ST2: Mirror(XY). Upside down, the stays in pairs on a mast are the Stuttgart fan on a trunk, with tension and compression swapped (Hooke's inversion).
ST3 → ST5: Rotate(roof, Y, θ=?; gravity). Tilt the roof plane while the loads stay vertical (about 15° in your earlier sketch).
ST1 → ST3: Branch(tip, k=2, halves, t=?) → Remove(middle twigs). Gather the bars in a left and a right group meeting at two branch nodes, keeping two twigs per branch.
KB6 → KB7: Rotate(X, θ=?). To confirm: Subsystem 3 read as Subsystem 2 tilted about X. Change the function if this reading is wrong.
KB7 → KB8: Superpose(S1, S2, S3). Overlay the three subsystems in one frame; shared nodes add their forces.
WI2 → WI3: PolarArray(Z, n=4). Turn the canopy section four times about the vertical axis, as you described.
WI3 → WI4: Intent(use: room to wait for the bus) ≈ Move(one side, +X, d=?). Stretch one side out so people waiting for the bus stay under the roof; the short side must then be tied down.
WI4 → WI5: Superpose(canopy, P4; tie-down). Stabilize: after the stretch the form and force diagrams no longer close, so a tension triangle from P4 is added as a tie-down.
WI5 → WI6: Extend(post, −Z, to ground) → Connect(ties). Stabilize: one more post and the last ties, until the canopy is in equilibrium again.
MA2 → WI1: Related, not a step: the Y-strut on a column (MA2) and the canopy on posts share the strut-and-post idea. To confirm.
P1 → T3a: Divide(sides, n=?) → Move(nodes, +Z; parabola). Divide the two sides and lift their nodes onto a parabola: a funicular arch over the tie.
T3 → IG3: Mirror(XY, copy) → Superpose(). Reflect the arch below the tie and keep both: a lens of two arches.
T3 → MA3: Mirror(XY). Hooke's inversion ("flip"): the arch hangs as a cable under a strut, which is the beam string MA3.
P2 → IG1: Extend(base, −Z, d=?). Stand the tent on legs from its two base corners.
IG5 → IG6: Intent(form: curved cables) ≈ Move(cable nodes, d=?; curve). Architectural intent: the straight tension cables become curves. Nearest function: move each cable's inner nodes off the straight line by a sag d.
HS1 → IG1: Related, not a step: IG1 uses the same Extend as HS1, at the base corners instead of the tip.
MA5 → MA6: Array(Y, n=?). Repeat the frame along the shed. Fill the number of bays.
SP3 → SP4a: Rotate(bars, Y, θ=?; gravity). One face of the box: the band with its bars turned to this face's angles while the loads stay vertical. Fill θ for this face.
SP3 → SP4b: Rotate(bars, Y, θ=?; gravity). One face of the box: the band with its bars turned to this face's angles while the loads stay vertical. Fill θ for this face.
SP3 → SP4c: Rotate(bars, Y, θ=?; gravity). One face of the box: the band with its bars turned to this face's angles while the loads stay vertical. Fill θ for this face.
SP3 → SP4d: Rotate(bars, Y, θ=?; gravity). One face of the box: the band with its bars turned to this face's angles while the loads stay vertical. Fill θ for this face.
IG1 → SP4e: Rotate(Z, θ=90°) → Superpose(cross). Cross the frame with a copy turned 90°, the same step as Khudi Bari KB1 → KB2.
SP4b → SP5: Superpose(walls ×4, roof; box). Fold the four walls round the square plan and put the crossed frames on top as the roof.
SP4c → SP5: Superpose(walls ×4, roof; box). Fold the four walls round the square plan and put the crossed frames on top as the roof.
SP4d → SP5: Superpose(walls ×4, roof; box). Fold the four walls round the square plan and put the crossed frames on top as the roof.
SP4e → SP5: Superpose(walls ×4, roof; box). Fold the four walls round the square plan and put the crossed frames on top as the roof.
SO2 → SO3: Superpose(SO1, SO2). The graded triangles become the web of the sloped beam string.
SO1 → SO3: Superpose(SO1, SO2). The graded triangles become the web of the sloped beam string.
MA3 → SO2: Rotate(Y, θ=?; gravity). Tilt the beam string to follow the slope while the loads stay vertical (your arrow pointed into MA3; it now starts from MA3).
P4 → SO1: Array(X, n=?) → Scale(s=?). Repeat the triangle along the span, each one scaled to the depth of the truss (your arrow pointed into P4; it now starts from P4).
SP4a → SP5: Superpose(walls ×4, roof; box). Fold the four walls round the square plan and put the crossed frames on top as the roof.
WI0 → WI4: Superpose(canopy, tie-down). Stabilize: the stretched canopy is tied down with the back-stayed cantilever WI0.
P3 → WI0: Move(one corner, −X, d=?) → Extend(corner, −Z, to ground). Make the V lopsided and tie its short side down to the ground: a back-stayed cantilever.
IG3 → IG4: Rotate(frames, X, θ=±?). Turn the two arches apart about the long axis, as Khudi Bari turns its two frames apart (KB4 → KB5). Fill θ.
IG2 → IG5: Superpose(spine frames, cable net). Put the frames along the spine and the cable net into one roof.
IG4 → IG5: Superpose(spine frames, cable net). Put the frames along the spine and the cable net into one roof.
IG1 → IG2: Array(Y, n=?) → Scale(s=?). Repeat the frame along the spine, each copy scaled so the heights follow the arch (largest in the middle).
MI2 → MI3: Array(X, n=7). Repeat the pylon seven times along the deck: the Millau Viaduct has seven piers.
P4 → WI5: Superpose(canopy, P4; tie-down). Stabilize: the tension triangle of P4 used as the canopy's tie-down.
T3a → T3: Connect(hangers). Hang the tie from the arch at every node: the tied arch, which is the arch-cable.
```
