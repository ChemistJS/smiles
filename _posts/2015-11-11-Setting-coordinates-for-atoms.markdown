---
layout: post
title:  "Setting coordinates for atoms"
date:   2015-11-11 00:20:00
categories: jekyll update
---

## How does the positioning of atoms work

The input to the positioning algorithm is a graph of the molecule.

Before positioning the graph must have established structures: coherent parts, biconnected components and rings. Rings are detected using an SSSR algorithm (smallest set of smallest rings [1]), which chooses the most "basic" rings from all the rings in the graph. Detection of this structures is fairly simple, so we assume it is done beforehand.

# Flipping the rings

This process facilitates the subsequent steps.

The ring arranged on the plane can be described in two ways: its components can be given in clockwise order or counterclockwise (this can also be thought of looking at a ring from the top or from the bottom). At this stage, objects describing the rings are converted to be consistent with each other - describe rings in the same direction. Absolute configuration does not matter here.

Algorithm can be described in this way: find two rings (`A`,` B`), which have a common edge (`x-y`) and then flip them so the ring `A` has a form `...xy...`, and the ring `B` has a form `...yx...` (or vice versa). Of course, care must be taken not to turn around the same rings all the time - in this implementation DFS on the rings is used, where the neighborhood is defined as having a common edge.

Note that for some molecules it is impossible to flip the rings to meet above requirements. This situation means that the molecule has a ring projecting above or below the plane. Examples of such molecules are: morphine, fullerene C60, cubane (C8H8). In the case of fullerene C60, cubane, and other compounds having the form of the cage seemingly the problem does not occur - it happens because the algorithm to find the rings (SSSR) does not detect all the walls of such structures as cycles.

Currently, in our implementation error in rings flipping is signaled by an exception.

	procedure flipDFS(ring):
		if ring was visited:
			return
		mark ring as visited
		flipped := false

		for each bond in ring:
			for each neighbour in (rings that contain bond):
				if neighbour != ring and neighbour was visited:
					if (direction of bond in ring) == (direction of bond in neighbour):
						if flipped:
							signal an error
						else
							flip ring
							flipped := true

		for each bond in ring:
			for each neighbour in (rings that contain bond):
				flipDFS(neighbour)

	mark all rings as not visited
	for each ring:
		flipDFS(ring)

# Greedy positioning of the rings

At this stage, each biconnected component is processed separately, and the coordinates of the vertices are calculated with respect to this component. For example, the vertex which is located at the junction of many biconnected components will have a number of different coordinates.

Short description of this step: having positioned a set of vertices we are trying to position the rest of vertices so that the all the rings found in the graph would be a regular polygons. One ring is considered at a time, we do not think about what comes next, and once the vertices are positioned they are never moved - this is a greedy algorithm.

The rings are processed with DFS, where neighborhood is seen as having a common edge. For each ring:

 * Generate a set of points which are the vertices of a regular polygon with the number of edges equal to the number of edges of the ring in question.
 * Calculate the optimal rotation and translation, so that the vertices of the regular polygon generated above, would be the closest (minimum sum of squares) to the matching vertices of the processed ring.
 * Apply calculated transformations to the vertices of the regular polygon.
 * Positions all vertices of the ring, which positions are not yet set, in locations which are transformed vertices of a regular polygon.

Within a single biconnected component the rings graph is connected, so all the rings of this component will be processed. Additionally, each vertex in biconnected component must be in at least one cycle (the only exceptions are components with one or two elements), so each vertex will have a set position.

This algorithm works well for molecules which rings within each biconnected component form a tree structure (without cycles), and molecules which cycles are not tensioned.

For particles that do not meet these conditions, the algorithm sets the position, however, visualization created based on them will not look good.

	function bestTransformation(matchSet):
		return transformaion t such that sum({abs(t(el[0]) - el[1])^2: el in matchSet}) is minimal

	procedure greedyDFS(ring):
		if ring was visited:
			return
		mark ring as visited

		biconnectedComponent := biconnected component that contains this ring
		regularPolygon := points of regular polygon of length same as ring length

		matchSet := []
		for i in [0 .. ring length - 1]:
			if ring.atoms[i] has position set inside biconnectedComponent:
				append (regularPolygon[i], ring.atoms[i] position inside biconnectedComponent) to matchSet
		transformation := bestTransformation(matchSet)

		for i in [0 .. ring length - 1]:
			if ring.atoms[i] has not position set inside biconnectedComponent:
				position of ring.atoms[i] inside biconnectedComponent := transformation(regularPolygon[i])

		for each bond in ring:
			for each neighbour in (rings that contain bond):
				greedyDFS(neighbour)

	mark all rings as not visited
	for each ring:
		greedyDFS(ring)

# Trivial biconnected components

At this point, in every biconnected component, which contains at least one cycle, the coordinates of all its vertices are set. Now the components without cycles are addressed. They may have one or two vertices, no more. We proceed as follows:

 * The component with one vertex: set its position within this component to `(0, 0)`
 * The component with two vertices: set their positions within this component to `(0, 0)` and `(d, 0)`

where `d` is the length of the edge.

	for each biconnectedComponent that has one atom:
		biconnectedComponent.atoms[0] position inside biconnectedComponent := (0, 0)
	for each biconnectedComponent that has two atoms:
		biconnectedComponent.atoms[0] position inside biconnectedComponent := (0, 0)
		biconnectedComponent.atoms[1] position inside biconnectedComponent := (d, 0)

# Thermal setting of the rings

This step is designed to improve imperfections of greedy algorithm positioning the rings, acting on the tensioned molecules. In short, the algorithm can be described as: allow the movement of vertices and add the forces that want to make a regular polygon from each ring.

The algorithm performs many iterations to improve position and interrupts the repetition when the solution ceases to differ widely. ("widely" is defined by a number.) A measure of difference between solutions is the average length of  vertex displacement that occurred during this step.

Vertex positions are always defined as positions within biconnected components. It can be imagined that every biconnected component is a different molecule. They will join later.

A single iteration for each ring:

 * Calculate the positions of the vertices of a regular polygon with the number of vertices equal to the number of vertices in processed ring
 * Move and rotate the polygon to obtain best alignment with the vertices of the processed ring (minimum sum of squares).
 * Save positions of transformed vertices of the polygon in arrays associated with the respective vertices of the graph

The result of these operations is to obtain, for each vertex, arrays of desired positions. From these tables average is calculated, and then position of the vertex is replaced by this average. The next iteration will operate on the new coordinates.

The average may be weighted - for example, if we want the smaller the rings to have a greater "priority of regularity" we can count positions in the smaller rings with a greater weight than those of large rings.

	procedure thermalStep(component):
		for each atom in component:
			atom.contributions := []

		for each ring in component:
			regularPolygon := points of regular polygon of length same as ring length

			matchSet := []
			for i in [0 .. ring length - 1]:
				if ring.atoms[i] has position set inside biconnectedComponent:
					append (regularPolygon[i], ring.atoms[i] position inside biconnectedComponent) to matchSet
			transformation := bestTransformation(matchSet)

			for i in [0 .. ring length - 1]:
				append (weight: 1, position: transformation(regularPolygon[i]) to ring.atoms[i].contributions

		for each atom in component:
			if atom.contributions length > 0:
				newPosition := average calculated from atom.contributions
				atom position inside component := newPosition

	while we decided to improve molecule layout:
		for each biconnectedComponent:
			thermalStep(biconnectedComponent)

# Aligning biconnected components

At this stage, we have set a position of each vertex inside each biconnected component to which it belongs. For positions to be useful in visualization, the coordinates should be aligned between the components. Alignment is understood as finding the translation vector and rotation angle, that must be applied to the coordinates inside the component to get absolute coordinates, for each component.

The starting point for finding the right transformation will be the nodes that belong to more than one component - let's call them junctions. There are two conditions:

 * Junction has one absolute position. This means that the coordinates of the junction in each of the components transformed by translation and rotation of the respective components must give the same point.
 * Junction represents a single atom, which in the drawing is to have a specified angles between its bonds. It imposes a condition on the rotation of the individual components.

Biconnected components within a single coherent piece of graph form a tree. It seems natural to process junctions in the order of traversing a tree.

In the current implementation, first in suffix (it has no meaning) order local transformations are determined - that is for the component `A` and some its child (in the sense of the tree) `B` a transformation is calculated. This transformation when applied to the component `B` will cause the junction to "fit" to the component `A`. Then, in the prefix order components are transformed according to the previously calculated values, except that the transformations are pushed down the tree and accumulated for the components to fit together globally.

	procedure computeTransformations(tree):
		for childTree in children of tree:
			computeTransformations(childTree)

		rootComponent := root component of tree
		for each atom in rootComponent:
			children := biconnected components that contains atom \ rootComponent
			if |children| > 0:
				compute transformations relative to rootComponent for children

	procedure performTransformations(tree, transformation):
		rootComponent := root component of tree
		rootAtoms := atoms of rootComponent
		apply transformation to rootAtoms

		for childTree in children of tree:
			relativeTransformation := transformation of childTree relative to rootComponent
			totalTransformation := compose transformation * relativeTransformation (relativeTransformation is applied first)
			performTransformations(childTree, totalTransformation)

	forest := set of trees of biconnected components in molecule

	for each tree in forest:
		computeTransformations(tree)
		performTransformations(tree, identity)

# Positioning connected parts

The final step is to move apart coherent subgraphs so they will be drawn in other places and will not overlap graphically. For this, any coherent part of the graph gets calculated maximal and minimal x and y coordinates reached by the vertices of the part. Then, based on the calculated values subgraphs are moved so they will be centered vertically and horizontally stacked one behind the other, with predetermined intervals in between.

	x := 0
	for each coherentComponent:
		calculate minx, maxx, miny, maxy for coherentComponent
		move all vertices of coherentComponent by (x - minx, -0.5 * (miny + maxy))
		x := maxx + spacing

# Bibliography

 * [1] Antonio Zamora, An Algorithm for Finding the Smallest Set of Smallest Rings, Journal of Chemical Information and Computer Sciences, 1976

There is also a version of this document in Polish [here]({{ site.baseurl }}{% post_url 2015-11-12-Setting-coordinates-polish-version %}).
