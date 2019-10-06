# When inserting float based AABBs, prefer NotNaN<> over OrderedFloat<>.

If you want to use this for floats, NotNaN has no overhead for comparisions, but has overhead for computation, the opposite is true for OrderedFloat.
For querying the tree colliding pairs, since no calculations are done, just a whole lot of comparisions, prefer NotNaN<>.
Other queries do require arithmatic, like raycast and knearest. So in those cases maybe ordered float is preferred.

