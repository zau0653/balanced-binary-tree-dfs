# balanced binary tree: how height-balance works, the O(n) DFS solution to LeetCode 110, and the mistakes that cost people the problem

Search "balanced binary tree" and you land in one of two places. Most people are staring at LeetCode 110, trying to work out why their solution passes the first example and fails the sixth. Others are reading a data structures chapter and running into AVL trees and red-black trees, wondering whether "balanced" is a property you check or something that maintains itself.

Both are the same idea seen from different angles, and the confusion between them is why the topic feels harder than it is. So let's separate them, then write the code that actually gets accepted.

## Two questions hiding behind one term

A balanced binary tree, in the loose textbook sense, is a tree where no leaf ends up much farther from the root than any other. The point of the shape is performance: keep the height near log n and lookups stay fast, instead of degrading into a linked list when you insert sorted data.

That definition splits into two very different tasks:

- **Checking** whether a given tree happens to be height-balanced. This is LeetCode 110. You get a tree, you return true or false, you change nothing.
- **Maintaining** balance while data changes. This is the AVL tree, red-black tree, 2-3 tree and B-tree family. Rotations happen on insert and delete, and the structure guarantees a height bound on its own. A red-black tree containing n nodes has height at most about 2·log(n+1), which is the proof behind those O(log n) operation times.

If an interviewer asks you to "check if a tree is balanced," they mean the first one. If they ask why Java's `TreeMap` doesn't degenerate, they mean the second. Mixing the two in an answer is a common way to sound like you've memorized a definition without understanding it.

There's also a naming trap: height-balanced is a property of the tree's shape, while "balanced binary search tree" describes a BST that maintains that shape as you mutate it. A perfectly height-balanced tree of random integers isn't necessarily a BST, and a BST isn't necessarily height-balanced.

## LeetCode 110 in plain terms

The problem gives you the root of a binary tree and asks whether it is height-balanced, defined as: for every node, the heights of its left and right subtrees differ by no more than 1.

Every node. Not just the root. That single word is where most failed submissions come from.

Quick examples of what that means in practice:

| Input | Result | Why |
| --- | --- | --- |
| `[]` | true | An empty tree is balanced |
| `[3,9,20,null,null,15,7]` | true | Every node's subtree heights differ by at most 1 |
| `[1,2,3,null,null,4]` | true | The deepest path and the shallowest differ by 1 at the root too |
| `[1,2,3,null,null,4,null,5]` | false | Node 3's right subtree reaches depth 2 while its left is empty |
| `[1,2,2,3,3,null,null,4,4]` | false | The imbalance appears deep in the tree, not at the root |

The target complexity is O(n) time, since you have to look at each node at least once, with O(n) space in the recursive version (the call stack, which is really O(h) where h is the tree height).

## The literal solution, and why it's O(n²)

The most direct translation of the definition computes a height for each node, compares, then recurses into both children:

python
class Solution:
    def isBalanced(self, root) -> bool:
        if root is None:
            return True
        left = self.height(root.left)
        right = self.height(root.right)
        if abs(left - right) > 1:
            return False
        return self.isBalanced(root.left) and self.isBalanced(root.right)

    def height(self, node) -> int:
        if node is None:
            return 0
        return 1 + max(self.height(node.left), self.height(node.right))


It's correct, it's easy to explain, and the complexity is worse than it looks. `height()` walks an entire subtree every time you call it, and you call it once per node. That's O(n²) on a skewed tree. For a small input it sails through; for a 1000-node path-shaped tree it's doing roughly half a million node visits to answer a yes/no question.

This version is worth writing in an interview anyway. Say it out loud, state the O(n²) cost, and then improve it. Interviewers usually care more about whether you notice the redundant work than whether you produce the optimal version immediately.

## Bottom-up DFS: height and balance in one traversal

The fix is to stop recomputing heights. Do a post-order traversal that returns the height of each subtree, and check balance at the same moment you learn those heights. To signal "this subtree is already broken" without a second return value, return -1, which is safe because a real height is never negative.

python
class Solution:
    def isBalanced(self, root) -> bool:
        return self.check(root) != -1

    def check(self, node) -> int:
        if node is None:
            return 0
        left = self.check(node.left)
        if left == -1:
            return -1
        right = self.check(node.right)
        if right == -1:
            return -1
        if abs(left - right) > 1:
            return -1
        return 1 + max(left, right)


Each node is visited exactly once, so the time is O(n). Space is O(h) for the recursion stack: O(log n) on a balanced tree, O(n) in the pathological case.

If a single return value feels too clever, returning a pair works just as well, and some people find it more readable:

python
def dfs(node):
    if node is None:
        return True, 0
    left_ok, left_h = dfs(node.left)
    right_ok, right_h = dfs(node.right)
    ok = left_ok and right_ok and abs(left_h - right_h) <= 1
    return ok, 1 + max(left_h, right_h)


The early `return -1` in the sentinel version is a real optimization, not just style: once one subtree is unbalanced, the whole tree is unbalanced, and you can stop climbing. The tuple version keeps computing heights all the way up.

👉 [Study the tree DFS pattern the way it's actually taught](https://algo.monster?fpr=li69) — AlgoMonster walks through height-balance, diameter, and the other post-order tree problems as one pattern instead of three unrelated tricks.

## Tracing `[1,2,3,null,null,4,null,5]`

This is the example that breaks naive solutions, so walk it once.

The root `1` has left child `2` and right child `3`. Node `3` has a right child `4`, and `4` has a right child `5`. Node `3`'s left subtree is empty, so its height is 0, while its right subtree has height 2. The difference of 2 makes node `3` unbalanced.

Now notice where the problem is. If you only compare the heights of the root's two subtrees, you get a difference of 1 and return true — wrong answer. The imbalance is three levels down. That's why the check has to run at every node, and why the bottom-up traversal is natural for this problem: it visits children before parents, so each node already knows its subtree heights when its turn comes.

## Bugs that show up again and again

**Checking only the root.** Covered above. The definition is recursive; the check has to be recursive.

**Returning a boolean from the recursive function and tracking height in a global.** It can be made to work, but you're now relying on mutation across test cases. This is exactly why almost every solution to this problem uses a helper function: the public method must return `bool`, while the recursion needs to return a height. Two different return types, two functions. The workaround is to stash heights in a dictionary keyed by node, an extra list argument, or — a genuinely bad idea — in the nodes' `val` fields.

**Forgetting to propagate the -1.** If you compute `abs(left - right) > 1` without checking `left == -1` first, you often get the right answer by accident, because `abs(-1 - something)` is frequently greater than 1. It breaks on the cases where it isn't. The early return is what makes the sentinel reliable.

**Mixing height conventions.** Leaf nodes have height 0 (one node, no edges) and an empty tree has height -1 by the standard definition, though most LeetCode solutions return 0 for `None` and count levels instead. Both conventions work. Switching between them mid-solution does not.

**Claiming O(n) while writing the O(n²) solution.** The nested `height()` calls are the tell. If you can point at the line that re-traverses, you'll catch it before the interviewer does.

**Recursion depth on skewed input.** A path-shaped tree pushes the stack to roughly n frames. Python's default recursion limit sits around 1000, which is close enough to the constraint ceiling on this problem that aggressively skewed test cases can surprise you. In practice the bottom-up solution returns early, so it rarely matters here, but it's worth knowing why the iterative version exists.

## What about the iterative version

You can do this with an explicit stack simulating post-order traversal, storing computed heights per node. It works and it avoids recursion limits. It's also more code, more places to be off by one, and slower in practice because of the manual bookkeeping. For a whiteboard interview, the recursive post-order is the answer; the iterative version is worth mentioning so the interviewer knows you understand the tradeoff, then dropping.

## Where this pattern reappears

Balanced Binary Tree is rarely asked in isolation. It's the same post-order thinking as:

- Maximum Depth of Binary Tree — the height function on its own
- Diameter of Binary Tree — return height, track the best left-plus-right sum in a variable
- Binary Tree Maximum Path Sum — same shape, harder bookkeeping
- Convert Sorted Array to Binary Search Tree — building a balanced tree instead of checking one
- Validate Binary Search Tree — different invariant, same "does this hold for every subtree" question
- Balance a Binary Search Tree — sorted array to balanced BST applied to an existing tree

If you can write the bottom-up height function without thinking, all of these get easier, because the hard part is the traversal shape, not the individual problem statements.

## Practicing the pattern instead of grinding problems

Here's the honest version of the usual advice. If your only goal is passing one interview next month, the code above plus a handful of tree problems is enough. If you keep getting stuck on tree questions, the gap usually isn't knowledge of trees — it's not recognizing quickly that a problem wants post-order DFS with a value returned upward, or pre-order with state passed down. That recognition is a pattern skill, and it comes from studying problems grouped by pattern rather than by difficulty.

AlgoMonster is built around that grouping. The curriculum is organized into pattern modules rather than a numbered problem list, it's written by engineers and competitive programmers rather than crowd-sourced, and each pattern comes with a template you're meant to reuse, plus diagrams for the recursion and traversal steps that are hard to follow in text. There's a Speedrun mode for timed review when you already know the pattern and just want to keep it sharp, and an AI assistant for when you're stuck mid-problem.

👉 [Check the current AlgoMonster plan pricing](https://algo.monster?fpr=li69) — prices move with promotions, so the checkout page is the only place worth trusting.

### Plans, as currently listed

| Plan | What you get | Price | Billing | Purchase |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample lessons and problems, enough to judge the teaching style | $0 | — | [Start free](https://algo.monster?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question sets updated monthly; access across the six courses; member discount on live bootcamps | $99/year, listed as $8.25/month billed annually; the subscribe page has shown a 3-day free trial before the annual charge | Annual, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro Lifetime | Everything in the annual plan, plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189, billed once | One-time | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

A few things worth knowing before you decide. There's no monthly option — it's annual or lifetime. Promotional pricing shifts: the subscribe page has at times displayed steeper limited-time offers (one variant showed $69.99/year and $89 lifetime), so the number you see at checkout may be lower than the list figure above. Third-party reviews also circulate stackable promo codes, but I can't verify any specific code is live right now, so treat those as something to try rather than something to count on.

Two bonus courses are bundled with a subscription: a CS foundations course and a build-an-LLM course. Specialty courses developed with outside partners, like quant interview prep, are sold separately. There's no team or enterprise plan — this is a self-study product.

### Who it fits, and who it doesn't

Reviews of AlgoMonster land on a consistent set of strengths and gaps. It assumes you can already write working code in one language; it teaches selection, meaning which structure and which traversal a problem is asking for. It's text and diagrams only, no videos, which is efficient if you learn by reading and a genuine dealbreaker if you don't. System design coverage is introductory, so senior candidates targeting L5 and above still need a dedicated resource for that round, and there's no live mock interview environment, so nothing there simulates the pressure of a 45-minute screen. The curriculum is designed to be worked through in roughly four to eight weeks at an hour or two a day.

For the specific problem in this article, the value is narrow and clear: you don't just get an accepted solution to LeetCode 110, you get the post-order DFS template that also solves depth, diameter, and path sum without re-deriving it each time.

## FAQ

**Is a height-balanced tree the same as a balanced BST?**
No. Height-balanced describes the tree's shape: subtree heights at every node differ by at most 1. A balanced BST is a search tree that keeps that shape while you insert and delete, using rotations. AVL and red-black trees are implementations of the second idea.

**Do I need the -1 sentinel?**
No. Returning a `(is_balanced, height)` pair is equally correct and easier to read. The sentinel version is slightly faster because it can stop traversing the moment it finds an imbalance.

**What's the time complexity of checking balance?**
O(n) bottom-up, O(n²) top-down with a separate height function. Both use O(h) stack space, which is O(log n) for balanced input and O(n) for a skewed tree.

**Is an empty tree balanced?**
Yes. `isBalanced(None)` should return true.

**Can I solve it with BFS or level-order traversal?**
You can, but it's awkward. Level order gives you depth from the root downward; the property you need is subtree height measured from the leaves upward, which is post-order's natural direction. Doing it with a queue means computing depth for every node separately, which puts you back at O(n²).

Learn the post-order height function once, use it everywhere. That's the whole point of the pattern.
