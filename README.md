# maximum product subarray: why the running minimum decides the answer, plus an O(n) DP traced on [2,3,-2,4], zeros, and double negatives

LeetCode 152 looks like the maximum subarray sum problem wearing a different hat. It isn't. You copy Kadane's algorithm, submit, and watch it fail on `[-2,0,-1]` or on some array with two negatives buried in it. The reason is simple and worth internalizing before you write a single line: with sums, extending a subarray never flips its sign. With products, one negative number can turn your worst running product into your best one.

That single fact is the whole problem. Everything else — the recurrence, the zeros, the edge cases — follows from it.

## The state you actually need

Most people's first instinct is to keep one running value: the best product ending at each position. That works for sums and breaks here. Once a negative number appears, the *smallest* product ending at the previous index becomes the most interesting candidate, because multiplying it by another negative produces a large positive.

So you track two values per index:

- `maxEndingHere` — the largest product of any subarray ending at the current index
- `minEndingHere` — the smallest product of any subarray ending at the current index

At each element `x`, every subarray ending at that position is either just `x`, or a subarray ending at the previous index extended by `x`. That gives three candidates:


maxEndingHere = max(x, prevMax * x, prevMin * x)
minEndingHere = min(x, prevMax * x, prevMin * x)


The standalone `x` matters more than people expect. It's the "start a new subarray here" option, and it's what makes zeros and all-negative arrays fall out correctly without a special branch.

Here's the whole thing in Python:

python
class Solution:
    def maxProduct(self, nums: list[int]) -> int:
        best = max_here = min_here = nums[0]

        for x in nums[1:]:
            prev_max, prev_min = max_here, min_here
            max_here = max(x, prev_max * x, prev_min * x)
            min_here = min(x, prev_max * x, prev_min * x)
            best = max(best, max_here)

        return best


And in Java, where the same bug — updating one variable before reading it for the other — shows up constantly:

java
class Solution {
    public int maxProduct(int[] nums) {
        int best = nums[0], maxHere = nums[0], minHere = nums[0];

        for (int i = 1; i < nums.length; i++) {
            int prevMax = maxHere, prevMin = minHere;
            maxHere = Math.max(nums[i], Math.max(prevMax * nums[i], prevMin * nums[i]));
            minHere = Math.min(nums[i], Math.min(prevMax * nums[i], prevMin * nums[i]));
            best = Math.max(best, maxHere);
        }

        return best;
    }
}


Both are one pass, constant extra space. The Java version is safe from overflow because the problem guarantees the product of any subarray fits in a 32-bit integer (`-10 <= nums[i] <= 10`, `1 <= nums.length <= 2 * 10^4`).

## Tracing it by hand, because the tables explain more than the code

Take the first official example, `[2,3,-2,4]`. Answer should be `6`, from `[2,3]`.

| Index | Value | prevMax | prevMin | Candidates (x, prevMax·x, prevMin·x) | maxEndingHere | minEndingHere | Best so far |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | 2 | — | — | — | 2 | 2 | 2 |
| 1 | 3 | 2 | 2 | 3, 6, 6 | 6 | 3 | 6 |
| 2 | −2 | 6 | 3 | −2, −12, −6 | −2 | −12 | 6 |
| 3 | 4 | −2 | −12 | 4, −8, −48 | 4 | −48 | 6 |

Notice what happened at index 2: the running maximum collapsed to `−2`, which looks like a dead end. It isn't. The `−12` sitting in `minEndingHere` is exactly the value that could explode into a large positive if another negative showed up. That's the reason the minimum has to be kept alive.

Now a messier one: `[3,2,0,5,6,-1]`. The best answer is `30`, from `[5,6]`.

| Index | Value | Candidates | maxEndingHere | minEndingHere | Best so far |
| --- | --- | --- | --- | --- | --- |
| 0 | 3 | — | 3 | 3 | 3 |
| 1 | 2 | 2, 6, 6 | 6 | 2 | 6 |
| 2 | 0 | 0, 0, 0 | 0 | 0 | 6 |
| 3 | 5 | 5, 0, 0 | 5 | 0 | 6 |
| 4 | 6 | 6, 30, 0 | 30 | 0 | 30 |
| 5 | −1 | −1, −30, 0 | 0 | −30 | 30 |

The zero at index 2 wiped the state without any conditional in the code. The candidates simply became `0` because `prevMax * 0` and `prevMin * 0` are both zero, and `0` was the largest. Then `[5,6]` built itself back up from index 3. If you ever find yourself writing a `if (nums[i] == 0) reset()` branch, you're doing extra work the recurrence already does for you.

## The bugs that actually cost people the interview

**Updating in the wrong order.** `maxHere = max(x, maxHere * x, minHere * x)` followed by `minHere = min(x, maxHere * x, ...)` uses the *new* `maxHere` in the min calculation. The code compiles, the examples pass, and one hidden test case fails. Save the previous values first, every time.

**Dropping the standalone `x`.** If you write `maxHere = max(prevMax * x, prevMin * x)` and initialize to `1`, a single-element array breaks and an array like `[-1,-2,-3]` returns something wrong. The comparison has to include `x` itself.

**Initializing to `1` or `0` instead of `nums[0]`.** Starting from `1` is a common habit borrowed from problems with non-negative inputs. It silently assumes the array has a positive product somewhere.

**Explaining only the maximum.** Interviewers listen for the word "minimum." If your explanation never mentions that a negative flips the sign, it reads as memorized rather than understood.

**Not mentioning zeros out loud.** The code handles zeros implicitly, but the interviewer wants to hear that a zero kills every product that spans it, so the array effectively splits into segments. Say it, even though you didn't write a branch for it.

## Complexity, and the alternatives worth knowing

| Approach | Time | Space | Notes |
| --- | --- | --- | --- |
| Brute force (all subarrays) | O(n²) | O(1) | There are n(n+1)/2 contiguous subarrays. With n up to 2·10⁴ that's roughly 2·10⁸ iterations — too slow in Python, marginal even in fast languages |
| One-pass max/min DP | O(n) | O(1) | The standard answer; a single scan with three variables |
| Prefix and suffix products | O(n) | O(1) | Multiply from the left and from the right, resetting the running product at each zero, and take the best of both. Two passes, same complexity |

The prefix/suffix version is worth having in your pocket. It's a little less intuitive but shorter to write under pressure, and it's a good way to show you know a second approach. Divide and conquer is technically possible here but almost never what an interviewer expects; the constraints and the contiguous structure point at the linear method.

One more framing that helps in the moment: instead of saying "I'll do dynamic programming," say "I'll track the maximum and minimum product ending at each index, and update the answer as I go." It's the same thing, described in a way that shows you know *why* the minimum exists.

## Why this problem is a pattern problem, not a one-off

Once you've solved 152 properly, you've learned a template: when a running state can flip sign or direction on the next input, carry both extremes. That same idea reappears in stock problems, in problems with negative multipliers, and in any DP where an optimal path might dip through a bad-looking state before recovering. Grinding forty random array problems will not teach you that; seeing the pattern name attached to the problem and to its relatives will.

This is where pattern-based prep earns its place. AlgoMonster publishes a written editorial for this exact problem — Maximum Product Subarray, LeetCode 152 — with runnable Python and Java solutions, the running max/min recurrence written out, and the complexity analysis, all as text and diagrams rather than video. That format matters if you're the kind of person who wants to scroll back to a specific line of reasoning instead of scrubbing a 20-minute video.

The platform's structure is deliberately narrow: a fixed set of reusable coding patterns, each with an explanation, a language-specific template, and practice problems attached to it. The subscription page lists the content as 500+ lessons across 6 courses, 1,000+ illustrations and 3,000+ editorials, real interview questions refreshed monthly, an unlimited AI assistant, and unlimited Speedrun practice sessions. Solutions and templates are available across multiple languages.

Worth stating plainly, since it's a real trade-off: AlgoMonster is text-and-diagram based. There are no video walkthroughs, so if you learn by watching someone talk through code, this is the wrong shape of product for you. Its problem bank is also far smaller than LeetCode's — you're paying for the pattern framework and the explanations, not for volume. For someone who already knows what to study and just wants more problems, that's a poor trade.

Here's the editorial page for this problem if you want to see how it's written before deciding anything: 👉 [Maximum Product Subarray editorial and solutions on AlgoMonster](https://algo.monster/liteproblems/152?fpr=li69).

## Plans and current pricing

AlgoMonster keeps its pricing to three options, with no month-to-month plan. Promotional pricing on the subscribe page changes with sales, which is why third-party write-ups quote figures that don't match each other — some captured $119/year and $459 lifetime, others $99 and $189. The checkout page is the only number that's ever current, so check it before you assume anything from a blog post, including this one.

| Plan | What's included | Price | Billing | Access |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems — enough to judge whether the text-and-diagram style works for you | $0 | — | [Start with the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; real company questions updated monthly; member discount on live bootcamps | $69.99/year (shown as $5.84/mo, billed annually) | Billed once a year, 12 months of full access | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan, plus lifetime updates on all current and future content | $89 | One-time payment | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

👉 [See the live prices on the AlgoMonster subscribe page](https://algo.monster/subscribe?fpr=li69) before you buy — promotional rates move, and the number that applies to you is the one at checkout.

Which one makes sense depends entirely on your timeline. If you're interviewing in the next two or three months, the annual plan covers you and costs less than a couple of months of most competing subscriptions. If you expect to be back on the interview circuit in a year or two — which is the common case for engineers who move companies every few years — the lifetime option is priced close enough to a single year that the decision isn't really a decision.

The free tier is genuinely useful for one specific thing: figuring out whether written explanations with diagrams are how you absorb algorithms. Fifteen minutes there will tell you more than any review, including this one. If you bounce off the format, you've spent nothing.

## Quick answers to the questions people actually search for

**Does the DP need arrays?** No. You need three variables: the running maximum, the running minimum, and the global best. Space is O(1).

**What if every number is negative?** The answer is a single-element subarray — the largest element. The standalone `x` candidate in the recurrence handles it, which is why dropping that candidate is such an expensive shortcut.

**Are zeros special-cased?** Not in the code. `0` becomes a candidate value, so the running max and min collapse to zero naturally and the array effectively restarts after it. Mention it in your explanation anyway, because interviewers use it as a signal that you've reasoned about the problem rather than recalled the solution.

**Is the brute force ever acceptable?** As a starting point in the interview, yes — stating it and then improving to O(n) is a normal progression. As a submission, no: at n = 2·10⁴, the quadratic loop will time out or come uncomfortably close.

The pattern is the takeaway here, not the specific array. Any time the next input can invert your running optimum, carry both extremes.
