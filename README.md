# Longest Palindromic Subsequence: The Recurrence, the O(n²) DP Table, and the Reverse-String Trick That Actually Works

`"bbbab"`. You stare at it, write down `"bbbb"`, length 4, and feel fine. Then someone asks for the longest palindromic *substring* of the same string and the answer is `"bbb"`, length 3. Two different algorithms, two different tables, and if you mix them up in an interview you will burn ten minutes discovering it.

The longest palindromic subsequence (LPS) problem is the cleaner of the two, and it's the one that teaches interval DP properly: a 2D table indexed by a start and an end index, filled in order of increasing interval length. Once that clicks, LeetCode 1143, 583, 1312 and 647 stop looking like separate problems.

This is the full walkthrough: the recurrence, why it's correct, three working implementations, the reverse-string shortcut and its one nasty trap, plus where to drill it if you want the pattern to stick.

## Subsequence vs substring: the distinction everything hinges on

A subsequence keeps the relative order of characters but is allowed to skip. A substring can't skip anything. That one difference changes the answer and the algorithm.

For `s = "bbbab"`:
- Longest palindromic subsequence: `"bbbb"`, length **4** (drop the `a`)
- Longest palindromic substring: `"bbb"`, length **3**

Another example from standard problem sets: `s = "agbdba"` gives `"abdba"`, length **5**. Same character set, different order, different answer.

A palindrome reads the same in both directions, so whatever you build has to be mirrored. The naive approach is to enumerate all 2ⁿ subsequences and test each one, which is exponential and dies somewhere around n = 25. Dynamic programming fixes that by noticing that the answer for `s[i..j]` is built entirely out of answers for smaller intervals.

The official constraint on LeetCode 516 is `1 <= s.length <= 1000`, letters lowercase only. That n ≤ 1000 is a hint: O(n²) time and O(n²) space is the intended target, and anything cubic will time out.

## The recurrence, and why each case is safe

Define `L(i, j)` as the length of the longest palindromic subsequence inside `s[i..j]`, inclusive. Four cases cover everything:

| Case | Condition | Value |
| --- | --- | --- |
| Empty interval | `i > j` | `0` |
| Single character | `i == j` | `1` |
| Matching ends | `s[i] == s[j]` | `2 + L(i+1, j-1)` |
| Non-matching ends | `s[i] != s[j]` | `max(L(i+1, j), L(i, j-1))` |

The matching-ends case is the one people trust too quickly. The claim is that if the two outermost characters are equal, there's always an optimal solution using both of them as the outer pair. Reason: if you have some palindrome inside `s[i..j]` that doesn't use both ends, you can wrap it with `s[i]` and `s[j]` and get something at least as long, because those two characters match each other and sit outside everything else. So pairing them costs you nothing.

When the ends don't match, neither can be paired with the other, so at least one of them is unused in any optimal solution. Dropping the left or dropping the right are the only two options, and you take the better one. This is exactly what allows the subsequence to be non-contiguous: you're permanently discarding characters as you shrink the interval, and the ones you discard on the way in can still be added on the outside higher up the recursion.

## Counting the work: why it lands at O(n²)

`L(i, j)` is identified by the pair `(i, j)` with `0 <= i <= j < n`. That's about `n(n+1)/2` distinct subproblems, and each one does constant work plus lookups. So the memoized or bottom-up solution is O(n²) time.

Without memoization you get the exponential blowup, because the recursion tree recomputes the same intervals over and over. `L(0, 5)` calls `L(0, 4)` and `L(1, 5)`, and both of those call `L(1, 4)`. That overlap is the whole reason DP exists here, and it's worth being able to say out loud in an interview rather than just writing the table.

## Three implementations that all work

### 1. Top-down with memoization

python
def longest_palindrome_subseq(s: str) -> int:
    n = len(s)
    memo = {}

    def best(i: int, j: int) -> int:
        if i > j:
            return 0
        if i == j:
            return 1
        if (i, j) in memo:
            return memo[(i, j)]

        if s[i] == s[j]:
            res = 2 + best(i + 1, j - 1)
        else:
            res = max(best(i + 1, j), best(i, j - 1))

        memo[(i, j)] = res
        return res

    return best(0, n - 1)


This mirrors the recurrence one-to-one, which makes it the right version to write first in an interview before optimizing. Python's recursion limit is a real concern at n = 1000 (roughly n deep on the worst chain), so either raise `sys.setrecursionlimit` or go bottom-up.

### 2. Bottom-up interval DP

python
def longest_palindrome_subseq(s: str) -> int:
    n = len(s)
    dp = [[0] * n for _ in range(n)]

    for i in range(n):
        dp[i][i] = 1

    for i in range(n - 1, -1, -1):
        for j in range(i + 1, n):
            if s[i] == s[j]:
                dp[i][j] = dp[i + 1][j - 1] + 2
            else:
                dp[i][j] = max(dp[i + 1][j], dp[i][j - 1])

    return dp[0][n - 1]


The loop order is the part that trips people. `dp[i][j]` needs `dp[i+1][j]`, `dp[i][j-1]` and `dp[i+1][j-1]`, all of which are shorter intervals. Iterating `i` from `n-1` down to `0` and `j` from `i+1` upward guarantees every dependency is already filled.

One detail that surprises people: for adjacent characters where `j == i + 1`, the expression `dp[i+1][j-1]` reads `dp[i+1][i]`, a cell below the main diagonal. Since the table is zero-initialized and that cell is never written, it returns 0, and `0 + 2 = 2` is the correct answer for a two-character matching pair. No special case needed, but you should know *why* it works rather than relying on luck.

### 3. Reverse the string and run LCS

python
def longest_palindrome_subseq(s: str) -> int:
    return lcs(s, s[::-1])  # standard 2-row O(n^2) LCS


It's a two-line answer if you already have LCS written. It's also the version most likely to be asked about, because interviewers know people copy it without understanding it.

Why it's valid: every palindromic subsequence of `s` is also a subsequence of `reverse(s)` (the reverse of a palindrome is itself), so LPS ≤ LCS(s, reverse(s)). The other direction takes a bit more work. Given a common subsequence, you match its characters from both ends and average two derived palindromes, which shows some palindrome is at least as long as the LCS. Both inequalities together give equality.

## The trap: this trick does not carry over to substrings

Swap "subsequence" for "substring" and the LCS trick silently produces wrong answers. Take `"racercar"`. Reverse it and you get `"racercar"`, whose common substrings with the original include `"rac"` and `"car"`. Neither is a palindrome. The longest common substring between a string and its reverse is not the longest palindromic substring.

The longest palindromic *substring* (LeetCode 5) needs either expand-around-center or Manacher's algorithm, and the counting variant (LeetCode 647) needs its own DP. If you can articulate this failure mode, you've demonstrated more understanding than most candidates who get 516 right.

## Mistakes that cost points

**Filling the table in the wrong order.** Iterating `j` on the outer loop and `i` on the inner loop works too, as long as all shorter intervals are done first. Iterating in plain row-major order from the top-left does not.

**Confusing the two problems.** "Palindromic subsequence" and "palindromic substring" appear in the same interview loop regularly, precisely because the answers differ.

**Returning the length when the question wants the string.** Length-only is LC 516. If you need the actual subsequence, either store the choice you made at each cell and backtrack, or reconstruct by walking from `dp[0][n-1]` inward, moving diagonally when `s[i] == s[j]` and jumping to the larger neighbor otherwise.

**Forgetting the `i > j` base case.** It shows up whenever a matching pair sits adjacent, and it's the difference between a correct answer and an index error.

**Assuming O(n²) space is required.** It isn't. `dp[i][j]` depends on row `i` and row `i+1`, so two rows of length n are enough: O(n²) time, O(n) space. Worth mentioning unprompted; it's the kind of follow-up interviewers ask when you finish early.

## Where this pattern reappears

Interval DP on a string shows up constantly once you can see it:

- **LC 516** Longest Palindromic Subsequence (this problem)
- **LC 1143** Longest Common Subsequence, the same table shape without the palindrome framing
- **LC 583** Delete Operation for Two Strings: `m + n - 2 × LCS`
- **LC 1312** Minimum Insertion Steps to Make a String Palindrome: exactly `n - LPS`
- **LC 5 / 647** Longest Palindromic Substring and Palindromic Substrings, different algorithms but the same "start index, end index" mental model

LC 1312 is the one worth internalizing. Every character not in the LPS has to be mirrored by an inserted character, so the answer falls straight out of the LPS table. Same code, different question.

## Practicing this without grinding 400 problems

The honest problem with DP is that it's not one pattern. Interval DP, knapsack, and sequence DP share vocabulary but not mechanics, and the way most people "learn" them is by solving 516, forgetting it in three weeks, and re-deriving it from scratch the day before an interview.

AlgoMonster is built around the opposite approach: each pattern gets an explanation, a reusable template, and then a small set of variations designed to make the template stick. Its free Lite Problems section has a write-up of 516 with the filled DP table and Python and Java solutions, which is a reasonable place to check your recurrence against someone else's before you start paying for anything. If you want to see how the interval-DP material fits into the wider curriculum, 👉 [browse AlgoMonster's pattern-based courses](https://algo.monster?fpr=li69).

Two things about the platform that are worth knowing before you buy, because they're real limitations rather than marketing complaints. There are no video walkthroughs at all; everything is text and interactive diagrams. And the problem count is in the low hundreds, not thousands, so it works as a pattern course but not as a volume gym. Third-party reviews consistently land on the same split: strong for learning the pattern, insufficient on its own for building speed. Plan on pairing it with LeetCode for reps.

The content is organized as 500+ lessons across 6 courses, with 1,000+ illustrations and 3,000+ editorials, plus an AI assistant and Speedrun mode for timed practice. Foundation and Build-an-LLM courses are bundled in with a Pro subscription rather than sold separately (the Quant Interview Prep course is the exception and is priced on its own).

## AlgoMonster plans and current pricing

AlgoMonster has no monthly option. It's free-tier, annual, or lifetime.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns, lessons and problems | $0 | No charge | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; company questions updated monthly; member discount on live bootcamps; Foundation and Build-an-LLM courses included | $99/year ($8.25/mo billed annually) | 12 months, cancel or upgrade anytime | [Get the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan, plus lifetime updates and permanent access with no renewals | $189 | One-time payment | [Get lifetime access](https://algo.monster/subscribe?fpr=li69) |

Two caveats on those numbers, because AlgoMonster runs aggressive and frequent sales. The subscribe page has displayed the annual plan as low as $69.99/year and the lifetime plan as low as $89 at different points, and third-party write-ups quote figures anywhere from $89 to $299/year and $99 to $459 for lifetime. Ignore all of it, including the numbers in this table, and treat the checkout page as the only source of truth. What doesn't move is the structure: one annual price, one lifetime price, no monthly billing, and lifetime covering future content additions.

For a decision rule based on the verified numbers: if you're prepping for interviews now and expect to be done in a few months, the annual plan is the lower-risk spend. If you expect to interview again in two or three years, the lifetime price is the better deal, and it stops mattering what the annual price does in the meantime.

## A short FAQ

**Is the answer always at least 1?** Yes, given the `n >= 1` constraint. Any single character is a palindrome of length 1, and `dp[i][i] = 1` sets that up.

**Can I do better than O(n²)?** Not for the general problem. There's no known subquadratic algorithm for exact LPS, which is why n ≤ 1000 in the constraints. (Longest palindromic *substring* does have a linear-time solution via Manacher's.)

**Does the reverse-plus-LCS approach give O(n) space for free?** Yes. A standard LCS with two rolling rows is O(n²) time and O(n) space, so if you already have that function, you get the space optimization without touching the palindrome logic.

**Which version should I write in an interview?** Memoized recursion first if the interviewer wants to see the reasoning, bottom-up if they care about the table and the space optimization. Know both, and know that `dp[i][j]` means "longest palindromic subsequence in `s[i..j]`" before you start typing, because explaining that definition clearly is most of the battle.

The recurrence is four lines. The hard part is the loop order, the base cases, and resisting the urge to reach for the LCS shortcut before you can derive the table yourself. 👉 [AlgoMonster's LPS problem walkthrough](https://algo.monster/problems/longest_palindromic_subsequence?fpr=li69) fills in the table step by step, which is a useful cross-check right after you've written yours.
