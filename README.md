# cs-maps-binary-trees-lab

## Learning goals 

1.  Review a `Map` implementation using a binary search tree (BST).
2.  Analyze the performance of BST methods.
3.  Explain the problem of unbalanced trees and how self-balancing trees solve this problem.


## Overview

This README presents solutions to the previous lab, then tests the performance of our tree-backed map.  We present a problem with our implementation and explain how Java's `TreeMap` solves this problem.


## Our version of `MyTreeMap`

In the previous lab we gave you the outline of `MyTreeMap` and asked you to fill in the missing methods.  Now we'll present our solutions, starting with `findNode`:


```java
	private Node findNode(Object target) {
		// some implementations can handle null as a key, but not this one
		if (target == null) {
        		throw new NullPointerException();
		}
		
		// something to make the compiler happy
		@SuppressWarnings("unchecked")
		Comparable<? super K> k = (Comparable<? super K>) target;
		
		// the actual search
        	Node node = root;
        	while (node != null) {
            		int cmp = k.compareTo(node.key);
            		if (cmp < 0)
                		node = node.left;
            		else if (cmp > 0)
                		node = node.right;
            		else
                		return node;
        	}
        	return null;
	}
```

`findNode` is a private methods used by `containsKey` and `get`; it is not part of the `Map` interface.  The parameter `target` is the key we're looking for.  We explained the first part of this function in the previous lab:

*  In this implementation, `null` is not a legal value for a key.

*  Before we can invoke `compareTo` on `target`, we have to typecast it to some kind of `Comparable`.  The "type wildcard" used here is as permissive as possible; that is, it works with any type that implements `Comparable` and whose `compareTo` method accepts `K` or any supertype of `K`.

After all that, the actual search is relatively simple.  We initialize a loop variable `node` so it refers to the root node.  Each time through the loop, we compare the target to `node.key`.  If the target is less than the current key, we move to the left child.  If it's greater, we move to the right child.  And if it's equal, we return the current node.

If we get to the bottom of the tree without finding the target, we conclude that it is not in the tree and return `null`.


## Searching for values

As we explained in the previous lab, the runtime of `findNode` is proportional to the height of the tree, not the number of nodes, because we don't have to search the whole tree.  But for `containsValue`, we have to search the values, not the keys; the BST property doesn't apply to the values, so we have to search the whole tree.

Our solution is recursive:

```java
	public boolean containsValue(Object target) {
		return containsValueHelper(root, target);
	}

	private boolean containsValueHelper(Node node, Object target) {
		if (node == null) {
			return false;
		}
		if (equals(target, node.value)) {
			return true;
		}
		if (containsValueHelper(node.left, target)) {
			return true;
		}
		if (containsValueHelper(node.right, target)) {
			return true;
		}
		return false;
	}
```

`containsValue` takes the target value as a parameter and immediately invokes `containsValueHelper`, passing the root of the tree as an additional parameter.

Here's how `containsValueHelper` works  

*  The first `if` statement checks the base case of the recursion.  If `node` is `null`, that means we have recursed to the bottom of the tree without finding the `target`, so we should return `false`.  Note that this only means that the target did not appear on one path through the tree; it is still possible that it will be found on another.

*  The second case checks whether we've found what we're looking for.  If so, we return `true`.  Otherwise, we have to keep going.

*  The third case makes a recursive call to search for `target` in the left subtree.  If we find it, we can return `true` immediately, without searching the right subtree.  Otherwise, we keep going.

*  The fourth case searches the right subtree.  Again, if we find what we are looking for, we return `true`.  Otherwise, having searched the whole tree, we return `false`.

This method "visits" every node in the tree, so it takes time proportional to the number of nodes.


## Implementing `put`

The `put` method is a little more complicated than `get` because it has to deal with two cases:  (1) If the given key is already in the tree, it replaces and returns the old value; (2) otherwise it has to add a new node to the tree, in the right place.

In the previous lab, we provided this starter code:

```java
	public V put(K key, V value) {
		if (key == null) {
			throw new NullPointerException();
		}
		if (root == null) {
			root = new Node(key, value);
			size++;
			return null;
		}
		return putHelper(root, key, value);
	}
```

And we asked you to fill in `putHelper`.  Here's our solution:

```java
	private V putHelper(Node node, K key, V value) {
		Comparable<? super K> k = (Comparable<? super K>) key;
		int cmp = k.compareTo(node.key);
		
		if (cmp < 0) {
			if (node.left == null) {
				node.left = new Node(key, value);
				size++;
				return null;
			} else {
				return putHelper(node.left, key, value);
			}
		}
		if (cmp > 0) {
			if (node.right == null) {
				node.right = new Node(key, value);
				size++;
				return null;
			} else {
				return putHelper(node.right, key, value);
			}
		}
		V oldValue = node.value;
		node.value = value;
		return oldValue;
	}
```

The first parameter, `node`, is initially the root of the tree, but each time we make a recursive call, it refers to a different subtree. 

As in `get`, we use the `compareTo` method to figure out which path to follow through the tree.  

If `cmp < 0 `, the key we're adding is less than `node.key`, so we want to look in the left subtree.  There are two cases:

*  If the left subtree is empty; that is, if `node.left` is `null`, we have reached the bottom of the tree without finding `key`.  At this point, we know that `key` isn't in the tree, and we know where it should go.  So we create a new node and add it as the left child of `node`.

*  Otherwise we make a recursive call to search the left subtree.

If `cmp > 0 `, the key we're adding is greater than `node.key`, so we want to look in the right subtree.  And we handle the same two cases as in the previous branch.

Finally, if `cmp == 0`, we found the key in the tree, so we replace and return the old value.

We wrote this method recursively to make it more readable, but it would be straightforward to re-write it iteratively, which you might want to do as an exercise.


## In-order traversal

The last method we asked you to write is `keySet`, which returns a `Set` that contains the keys from the tree in ascending order.  In other implementations of `Map`, the keys returned by `keySet` are in no particular order, but one of the capabilities of the tree implementation is that it is simple and efficient to sort the keys.  So we should take advantage of that.

Here's our solution:

```java
	public Set<K> keySet() {
		Set<K> set = new LinkedHashSet<K>();
		addInOrder(root, set);
		return set;
	}

	private void addInOrder(Node node, Set<K> set) {
		if (node == null) return;
		addInOrder(node.left, set);
		set.add(node.key);
		addInOrder(node.right, set);		
	}
```

In `keySet`, we create a `LinkedHashSet`, which is a `Set` implementation that keeps the elements in order (unlike most other `Set` implementations).  Then we call `addInOrder` to traverse the tree.

The first parameter, `node`, is initially the root of the tree, but as you should expect by now, we use it to traverse the tree recursively.  `addInOrder` performs a classic "in-order traversal" of the tree.

If `node` is `null`, that means the subtree is empty, so we return without adding anything to `set`.  Otherwise we:

1.  Traverse the left subtree.
2.  Add `node.key`.
3.  Traverse the right subtree.

Remember that the BST property guarantees that all nodes in the left subtree are less than `node.key`, and all nodes in the right subtree are greater.  So we know that `node.key` has been added in the correct order.

Applying the same argument recursively, we know that the elements from the left subtree are in order, as well as the elements from the right subtree.

And the base case is correct: if the subtree is empty, no keys are added.

So we can conclude that this method adds all keys in the correct order.

Because this method visits every node in the tree, like `containsValue`, it takes time proportional to `n`.


## The logarithmic methods

In `MyTreeMap`, the methods `get` and `put` take time proportional to the height of the tree, `h`.  In the previous lab, we showed that if the tree is full — if every level of the tree contains the maximum number of nodes — the height of the tree is proportional to `log n`.

And we claimed that `get` and `put` are logarithmic; that is, they take time proportional to `log n`.  But for most applications, there's no guarantee that the tree is full.  In general, the shape of the tree depends on the keys and what order they are added.

To see how this works out in practice, we'll test our implementation with two sample datasets: a list of random strings and a list of timestamps in increasing order.

Here's the code that generates random strings:

```java
		Map<String, Integer> map = new MyTreeMap<String, Integer>();
		
		for (int i=0; i<n; i++) {
			String uuid = UUID.randomUUID().toString();
			map.put(uuid, 0);
		}
```

`UUID` is a class in the `java.util` package that can generate a random "universally unique identifier".  UUIDs are useful for a variety of applications, but in this example we're taking advantage of an easy way to generate random strings.

We ran this code with `n=16384` and measured the runtime and the height of the final tree.  Here's the output:

    Time in milliseconds = 151
    Final size of MyTreeMap = 16384
    log base 2 of size of MyTreeMap = 14.0
    Final height of MyTreeMap = 33

We computed "log base 2 of size of MyTreeMap" to see how tall the tree would be if it were full.  The result indicates that a full tree with height 14 would contain 16,384 nodes.

The actual tree of random strings has height 33, which is substantially more than the theoretical minimum, but not too bad.  To find one key in a collection of 16,384, we only have to make 33 comparisons.  Compared to a linear search, that's almost 500 times faster.

This performance is typical with random strings or other keys that are added in no particular order.  The final height of the tree might be 2-3 times the theoretical minimum, but it is still proportional to `log n`, which is much less than `n`.  In fact, `log n` grows so slowly as `n` increases, it can be difficult to distinguish logarithmic time from contant time in practice.

However, binary search trees don't always behave so well.  Let's see what happens when we add keys in increasing order.  Here's an example that measures timestamps in nanoseconds and uses them as keys:

```java
		MyTreeMap<String, Integer> map = new MyTreeMap<String, Integer>();
		
		for (int i=0; i<n; i++) {
			String timestamp = Long.toString(System.nanoTime());
			map.put(timestamp, 0);
		}
```

`System.nanoTime` returns an integer with type `long` that indicates elapsed time in nanoseconds.  Each time we call it, we get a somewhat bigger number.  When we convert these timestamps to strings, they appear in increasing alphabetical order.

And let's see what happens when we run it:

    Time in milliseconds = 1158
    Final size of MyTreeMap = 16384
    log base 2 of size of MyTreeMap = 14.0
    Final height of MyTreeMap = 16384

The runtime is longer than in the previous case, more than 7 times longer.  If you wonder why, take a look at the final height of the tree: 16384!

If you think about how `put` works, you can figure out what's going on.  Every time we add a new key, it's larger than all of the keys in the tree, so we always choose the right subtree, and always add the new node as the right child of the rightmost node.  The result is an "unbalanced" tree that only contains right children.

The height of this tree is proportional to `n`, not `log n`, so the performance of `get` and `put` is linear, not logarithmic.


## Self-balancing trees

There are two possible solutions to this problem:

*  You could avoid adding keys to the `Map` in order.  But this is not always possible.

*  You could make a tree that does a better job of handling keys if they happen to be in order.

The second solution is better, and there are several ways to do it.  The most common is to modify `put` so that it detects when the tree is starting to become unbalanced and, if so, rearranges the nodes.  Trees with this capability are called "self-balancing".  Common self-balancing trees include the AVL tree ("AVL" are the initials of the inventors), and the red-black tree, which is what the Java `TreeMap` uses.

In our example code, if we replace `MyTreeMap` with the Java `TreeMap`, the runtimes are about the same for the random strings and the timestamps.  In fact, the timestamps run faster, even though they are in order, probably because they take less time to hash.

In summary, a binary search tree can implement `get` and `put` in logarithmic time, but only if the keys are added in an order that keeps the tree sufficiently balanced.  Self-balancing trees avoid this problem by doing some additional work each time a new key is added.


## One more exercise

In the previous lab we decided not to implement `remove`, but you might want to try it.  If you remove a node from the middle of the tree, you have to rearrange the remaining nodes to restore the BST property.  You can probably figure out how to do that on your own, or you can [read the explanation here](https://en.wikipedia.org/wiki/Binary_search_tree#Deletion).

Removing a node and rebalancing a tree are similar operations: if you do this exercise, you will have a better idea of how self-balancing trees work.


## Resources

*  [The documentation of `TreeMap`](https://docs.oracle.com/javase/7/docs/api/java/util/TreeMap.html) provides more information about the implementation.

*  You can [read more about red-black trees here](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree).
