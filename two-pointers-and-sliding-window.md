# Two Pointers & Sliding Window

## The Problem That Started It All

Imagine you have a sorted array and someone asks you:

> "Find two numbers that add up to a target."

Your first instinct — the brute force — is to try every pair:

```cpp
#include <vector>
using namespace std;

// O(n^2) — checks every possible pair
bool hasPairWithSum(vector<int>& nums, int target) {
    for (int i = 0; i < nums.size(); i++) {
        for (int j = i + 1; j < nums.size(); j++) {
            if (nums[i] + nums[j] == target)
                return true;
        }
    }
    return false;
}
```

This works. But it's `O(n^2)`. For `n = 100,000` that's 10 billion operations. In competitive programming, you typically get about `10^8` operations per second. That's 100 seconds. Time limit is usually 1–2 seconds.

You need something better.

---

## Before We Go Further: C++ Things You Need to Know

If you're coming from C, a few things in the code above might look unfamiliar. Let's address them now so they don't distract us later.

### `vector<int>`

In C, you'd use a raw array: `int arr[100]`. The problem is you need to know the size at compile time (or use `malloc`). In C++, `vector` is a dynamic array that grows as needed.

```cpp
#include <vector>
using namespace std;

vector<int> v;         // empty vector
v.push_back(10);       // v is now {10}
v.push_back(20);       // v is now {10, 20}
v.size();              // returns 2
v[0];                  // returns 10 — same indexing as arrays
```

Think of `vector<int>` as "an array of ints that manages its own memory." The `<int>` part is a **template** — it tells the compiler what type this vector holds. `vector<string>`, `vector<double>`, etc.

### `using namespace std;`

Everything in the C++ standard library lives inside a namespace called `std`. Without `using namespace std;`, you'd write `std::vector<int>`, `std::string`, `std::cout`, etc. The `using` directive lets you drop the `std::` prefix. In competitive programming, everyone uses it. In production code, people avoid it. We're doing competitive programming.

### References: `vector<int>&`

In C, if you pass an array to a function, you pass a pointer. In C++, if you pass a `vector<int>` to a function **by value**, it copies the entire vector. That's expensive. The `&` makes it a **reference** — an alias to the original. No copy.

```cpp
void modify(vector<int>& v) {
    v[0] = 999;  // modifies the original
}

void readonly(const vector<int>& v) {
    // can read v but can't modify it
    // 'const' is a promise: "I won't change this"
}
```

### Range-based for loops

C++ has a cleaner way to iterate:

```cpp
vector<int> nums = {1, 2, 3, 4, 5};

// C-style
for (int i = 0; i < nums.size(); i++) {
    cout << nums[i] << " ";
}

// C++ range-based for
for (int x : nums) {
    cout << x << " ";
}

// If you need to modify elements:
for (int& x : nums) {
    x *= 2;  // doubles each element in-place
}
```

### `auto`

C++ can infer types:

```cpp
auto x = 42;          // int
auto s = string("hi"); // string
auto it = v.begin();   // vector<int>::iterator
```

In competitive programming, `auto` saves typing. Use it when the type is obvious from context.

---

## The Two-Pointer Idea

Back to our problem. The array is **sorted**. That's the key insight we haven't used.

Consider `[1, 3, 5, 7, 9]` with target `10`.

Place one pointer at the start, one at the end:

```
 L              R
[1, 3, 5, 7, 9]
 sum = 1 + 9 = 10   ← found it!
```

But what if the target were `8`?

```
 L              R
[1, 3, 5, 7, 9]
 sum = 1 + 9 = 10   → too big, move R left

 L           R
[1, 3, 5, 7, 9]
 sum = 1 + 7 = 8    ← found it!
```

And if the target were `12`?

```
 L              R
[1, 3, 5, 7, 9]
 sum = 1 + 9 = 10   → too small, move L right

    L           R
[1, 3, 5, 7, 9]
 sum = 3 + 9 = 12   ← found it!
```

**Why does this work?**

- If the sum is too big, moving `R` left makes it smaller (array is sorted).
- If the sum is too small, moving `L` right makes it bigger.
- Each step eliminates an entire row/column of the pair search space.

Here's the code:

```cpp
#include <iostream>
#include <vector>
using namespace std;

// O(n) — each pointer moves at most n times
pair<int, int> twoSum(vector<int>& nums, int target) {
    int left = 0;
    int right = nums.size() - 1;

    while (left < right) {
        int sum = nums[left] + nums[right];

        if (sum == target)
            return {left, right};  // C++11: brace initialization of pair
        else if (sum < target)
            left++;
        else
            right--;
    }

    return {-1, -1};  // not found
}

int main() {
    vector<int> nums = {1, 3, 5, 7, 9, 11};
    auto [i, j] = twoSum(nums, 12);  // C++17: structured bindings

    if (i != -1)
        cout << "Found: " << nums[i] << " + " << nums[j] << endl;
    else
        cout << "No pair found" << endl;

    return 0;
}
```

### C++ note: `pair<int, int>` and structured bindings

`pair<int, int>` holds two values. You access them with `.first` and `.second`:

```cpp
pair<int, int> p = {3, 7};
cout << p.first;   // 3
cout << p.second;  // 7
```

The `auto [i, j] = ...` syntax (C++17) unpacks a pair into two variables. It's called a **structured binding**. Older code would write:

```cpp
pair<int, int> result = twoSum(nums, 12);
int i = result.first;
int j = result.second;
```

---

## Why Two Pointers Went From O(n^2) to O(n)

The brute force tries all `n*(n-1)/2` pairs. Two pointers tries at most `n` steps — each step either advances `left` or retreats `right`, and neither pointer ever reverses. Total pointer movement is bounded by `n`.

This is the core insight: **when the data has structure (sorted, partitioned, etc.), you can use that structure to skip large chunks of the search space.**

---

## Two Pointers: Variant 2 — Same Direction (The Slow-Fast Pattern)

Not all two-pointer problems use one pointer at each end. Sometimes both pointers start at the beginning and move in the same direction, but at different speeds.

### Problem: Remove Duplicates from Sorted Array

Given a sorted array, remove duplicates **in-place** and return the new length.

```
Input:  [1, 1, 2, 3, 3, 3, 4]
Output: [1, 2, 3, 4, _, _, _]  → return 4
```

The idea: use a **slow** pointer to track where the next unique element should go, and a **fast** pointer to scan through the array.

```cpp
int removeDuplicates(vector<int>& nums) {
    if (nums.empty()) return 0;

    int slow = 0;  // points to the last unique element placed

    for (int fast = 1; fast < nums.size(); fast++) {
        if (nums[fast] != nums[slow]) {
            slow++;
            nums[slow] = nums[fast];
        }
    }

    return slow + 1;  // length of unique portion
}
```

Trace through `[1, 1, 2, 3, 3, 3, 4]`:

```
fast=1: nums[1]=1 == nums[0]=1  → skip
fast=2: nums[2]=2 != nums[0]=1  → slow=1, nums[1]=2  → [1,2,2,3,3,3,4]
fast=3: nums[3]=3 != nums[1]=2  → slow=2, nums[2]=3  → [1,2,3,3,3,3,4]
fast=4: nums[4]=3 == nums[2]=3  → skip
fast=5: nums[5]=3 == nums[2]=3  → skip
fast=6: nums[6]=4 != nums[2]=3  → slow=3, nums[3]=4  → [1,2,3,4,3,3,4]
return 4
```

The first 4 elements are `[1, 2, 3, 4]`. We don't care about what's after — the function tells the caller "only the first 4 elements matter."

### Why two pointers here?

You could create a new array and copy unique elements into it, but that uses `O(n)` extra space. The two-pointer approach does it **in-place** with `O(1)` extra space. In competitive programming, space constraints matter.

---

## Two Pointers: Variant 3 — Partitioning

### Problem: Move all zeros to the end

```
Input:  [0, 1, 0, 3, 12]
Output: [1, 3, 12, 0, 0]
```

This is the same slow-fast pattern with a twist:

```cpp
void moveZeroes(vector<int>& nums) {
    int slow = 0;  // where the next non-zero should go

    for (int fast = 0; fast < nums.size(); fast++) {
        if (nums[fast] != 0) {
            swap(nums[slow], nums[fast]);
            slow++;
        }
    }
}
```

`swap` is a C++ standard library function — it exchanges two values. In C, you'd write:

```c
int temp = a;
a = b;
b = temp;
```

In C++: `swap(a, b);`. Clean.

Trace through `[0, 1, 0, 3, 12]`:

```
fast=0: 0 → skip
fast=1: 1 → swap(nums[0], nums[1]) → [1, 0, 0, 3, 12], slow=1
fast=2: 0 → skip
fast=3: 3 → swap(nums[1], nums[3]) → [1, 3, 0, 0, 12], slow=2
fast=4: 12 → swap(nums[2], nums[4]) → [1, 3, 12, 0, 0], slow=3
```

This is essentially the **partition** step from quicksort: separate elements into two groups based on a condition.

---

## Recognizing Two-Pointer Problems

Two pointers work when:

1. **The data is sorted** (or can be sorted without losing information).
2. **You're searching for pairs/triplets** that satisfy some condition.
3. **You need to do in-place operations** with O(1) space.
4. **You can eliminate search space** by moving one pointer based on a comparison.

If you see any of these signals, think two pointers.

---

## The Three Sum Problem: Two Pointers in Disguise

This is a classic. Given an array, find all unique triplets that sum to zero.

The trick: sort the array, fix one element, then run two-sum on the remainder.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>  // for sort()
using namespace std;

vector<vector<int>> threeSum(vector<int>& nums) {
    vector<vector<int>> result;
    sort(nums.begin(), nums.end());  // O(n log n)

    for (int i = 0; i < (int)nums.size() - 2; i++) {
        // Skip duplicate values for i
        if (i > 0 && nums[i] == nums[i - 1]) continue;

        int left = i + 1;
        int right = nums.size() - 1;
        int target = -nums[i];

        while (left < right) {
            int sum = nums[left] + nums[right];

            if (sum == target) {
                result.push_back({nums[i], nums[left], nums[right]});

                // Skip duplicates for left and right
                while (left < right && nums[left] == nums[left + 1]) left++;
                while (left < right && nums[right] == nums[right - 1]) right--;

                left++;
                right--;
            } else if (sum < target) {
                left++;
            } else {
                right--;
            }
        }
    }

    return result;
}

int main() {
    vector<int> nums = {-1, 0, 1, 2, -1, -4};
    auto triplets = threeSum(nums);

    for (auto& triplet : triplets) {
        cout << "[";
        for (int i = 0; i < 3; i++) {
            cout << triplet[i];
            if (i < 2) cout << ", ";
        }
        cout << "]" << endl;
    }
    // Output:
    // [-1, -1, 2]
    // [-1, 0, 1]
    return 0;
}
```

### C++ note: `sort()` and iterators

`sort(nums.begin(), nums.end())` sorts a vector in-place. `begin()` and `end()` return **iterators** — think of them as generalized pointers. `begin()` points to the first element, `end()` points one **past** the last element (a half-open range `[begin, end)`).

In C, you'd use `qsort()` with a comparison function pointer. C++'s `sort` is faster (it uses introsort), type-safe, and simpler to call.

### C++ note: `(int)nums.size() - 2`

`nums.size()` returns `size_t`, which is an **unsigned** type. If the vector is empty, `size() - 2` underflows to a huge number (like 18446744073709551614). Casting to `int` prevents this. This is a common trap in competitive programming. Always cast `size()` when doing subtraction.

### Complexity

- Sorting: O(n log n)
- Outer loop: O(n)
- Inner two-pointer: O(n) per iteration

Total: O(n^2). Brute force would be O(n^3). We saved an order of magnitude.

---

## Transition: From Two Pointers to Sliding Window

Two pointers on a sorted array let us search efficiently. But what about problems on **unsorted** arrays where we care about **contiguous subarrays** (a.k.a. windows)?

> "Find the longest substring without repeating characters."
>
> "Find the smallest subarray with sum >= target."

These aren't pair-search problems. These are about finding an **optimal contiguous range**. This is where the sliding window comes in.

The sliding window is really just two pointers moving in the same direction, maintaining a "window" between them. But the *mental model* is different enough that it deserves its own treatment.

---

## The Sliding Window Idea

Imagine you have a long strip of paper with numbers on it, and you're looking at it through a small window (a rectangular hole in a piece of cardboard). You can slide the window left and right.

```
Array: [2, 1, 5, 1, 3, 2]

Window of size 3:
  [2, 1, 5] 1, 3, 2    → sum = 8
  2, [1, 5, 1] 3, 2    → sum = 7
  2, 1, [5, 1, 3] 2    → sum = 9  ← max
  2, 1, 5, [1, 3, 2]   → sum = 6
```

### Fixed-Size Sliding Window

The simplest variant: the window has a fixed size `k`, and you slide it across the array.

**Problem: Find the maximum sum of any subarray of size k.**

Brute force: for each starting position, sum `k` elements. O(n*k).

Sliding window insight: when you slide the window one step right, you **add** the new element entering from the right and **subtract** the element leaving from the left. You don't recompute the entire sum.

```cpp
#include <iostream>
#include <vector>
#include <climits>  // for INT_MIN
using namespace std;

int maxSubarraySum(vector<int>& nums, int k) {
    // First, compute the sum of the initial window
    int windowSum = 0;
    for (int i = 0; i < k; i++) {
        windowSum += nums[i];
    }

    int maxSum = windowSum;

    // Slide the window: add the new right element, remove the old left element
    for (int i = k; i < nums.size(); i++) {
        windowSum += nums[i];       // new element enters the window
        windowSum -= nums[i - k];   // old element leaves the window
        maxSum = max(maxSum, windowSum);
    }

    return maxSum;
}

int main() {
    vector<int> nums = {2, 1, 5, 1, 3, 2};
    cout << maxSubarraySum(nums, 3) << endl;  // 9
    return 0;
}
```

This is O(n). Each element is added once and removed once.

### C++ note: `max()` and `<climits>`

`max(a, b)` returns the larger of two values. It's in `<algorithm>`, but most competitive programmers include it transitively through other headers. `INT_MIN` is the smallest possible `int` value (usually -2147483648), defined in `<climits>` (C++'s version of C's `<limits.h>`).

---

## Variable-Size Sliding Window

This is where things get truly powerful. The window doesn't have a fixed size — it **expands** and **contracts** based on some condition.

The general pattern:

```
left = 0
for right = 0 to n-1:
    add nums[right] to the window

    while (window violates some condition):
        remove nums[left] from the window
        left++

    update the answer
```

Both `left` and `right` only move forward. Total work: O(n).

### Problem: Smallest Subarray with Sum >= Target

Given an array of positive integers and a target, find the length of the smallest contiguous subarray whose sum is >= target.

```
Input:  nums = [2, 3, 1, 2, 4, 3], target = 7
Output: 2  (subarray [4, 3])
```

```cpp
#include <iostream>
#include <vector>
#include <climits>
using namespace std;

int minSubarrayLen(int target, vector<int>& nums) {
    int left = 0;
    int windowSum = 0;
    int minLen = INT_MAX;

    for (int right = 0; right < nums.size(); right++) {
        windowSum += nums[right];  // expand window

        // Contract window while the condition is satisfied
        while (windowSum >= target) {
            minLen = min(minLen, right - left + 1);
            windowSum -= nums[left];  // shrink from left
            left++;
        }
    }

    return (minLen == INT_MAX) ? 0 : minLen;
}

int main() {
    vector<int> nums = {2, 3, 1, 2, 4, 3};
    cout << minSubarrayLen(7, nums) << endl;  // 2
    return 0;
}
```

Let's trace through this carefully:

```
right=0: sum=2,  < 7
right=1: sum=5,  < 7
right=2: sum=6,  < 7
right=3: sum=8,  >= 7 → minLen=4, shrink: sum=8-2=6, left=1. Now 6 < 7, stop.
right=4: sum=10, >= 7 → minLen=4, shrink: sum=10-3=7, left=2.
                  >= 7 → minLen=3, shrink: sum=7-1=6, left=3. Now 6 < 7, stop.
right=5: sum=9,  >= 7 → minLen=2, shrink: sum=9-2=7, left=4.
                  >= 7 → minLen=2, shrink: sum=7-4=3, left=5. Now 3 < 7, stop.

Answer: 2
```

**Why is this O(n) and not O(n^2)?** It looks like there's a nested loop. But notice: `left` only moves forward. Over the entire execution, `left` goes from 0 to at most n. So the inner `while` loop runs at most n times **total** across all iterations of the outer `for` loop. Each element is added to the window once and removed at most once. Total operations: 2n = O(n).

This "amortized O(n)" reasoning is crucial for sliding window problems. Don't be fooled by nested loops.

---

## Sliding Window with a Hash Map

### Problem: Longest Substring Without Repeating Characters

Given a string, find the length of the longest substring without any repeating characters.

```
Input:  "abcabcbb"
Output: 3  ("abc")
```

Now we need to track **what's inside the window**, not just a running sum. We use a hash map (or hash set).

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
using namespace std;

int lengthOfLongestSubstring(string s) {
    unordered_map<char, int> charIndex;  // char → last seen index
    int maxLen = 0;
    int left = 0;

    for (int right = 0; right < s.size(); right++) {
        char c = s[right];

        // If this char is already in the window, jump left past it
        if (charIndex.count(c) && charIndex[c] >= left) {
            left = charIndex[c] + 1;
        }

        charIndex[c] = right;
        maxLen = max(maxLen, right - left + 1);
    }

    return maxLen;
}

int main() {
    cout << lengthOfLongestSubstring("abcabcbb") << endl;  // 3
    cout << lengthOfLongestSubstring("bbbbb") << endl;     // 1
    cout << lengthOfLongestSubstring("pwwkew") << endl;    // 3 ("wke")
    return 0;
}
```

### C++ note: `unordered_map`

`unordered_map<Key, Value>` is a hash table. Average O(1) lookup, insert, delete.

```cpp
unordered_map<string, int> ages;
ages["alice"] = 30;          // insert
ages["bob"] = 25;
ages.count("alice");         // 1 (exists) or 0 (doesn't)
ages["alice"];               // 30
ages.erase("alice");         // remove

// Iterate over all entries
for (auto& [key, value] : ages) {
    cout << key << ": " << value << endl;
}
```

There's also `map<Key, Value>` which uses a balanced BST (red-black tree). O(log n) operations, but keys are sorted. Use `unordered_map` when you don't need sorted keys (which is most of the time in CP).

### C++ note: `string`

In C, strings are `char*` arrays with a null terminator. In C++, `string` is a proper object:

```cpp
string s = "hello";
s.size();        // 5
s[0];            // 'h'
s += " world";   // concatenation: "hello world"
s.substr(0, 5);  // "hello" — substring from index 0, length 5
```

You can iterate over characters with `for (char c : s)`.

---

## Sliding Window with a Frequency Counter

### Problem: Minimum Window Substring

Given strings `s` and `t`, find the smallest substring of `s` that contains all characters of `t` (including duplicates).

```
Input:  s = "ADOBECODEBANC", t = "ABC"
Output: "BANC"
```

This is one of the hardest standard sliding window problems. Let's build up to it.

The approach:
1. Count what we need (frequency of each char in `t`).
2. Expand the window to the right until we have everything.
3. Contract from the left to find the minimum window.
4. Repeat.

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <climits>
using namespace std;

string minWindow(string s, string t) {
    if (t.empty() || s.empty()) return "";

    // Count required characters
    unordered_map<char, int> need;
    for (char c : t) need[c]++;

    int required = need.size();  // number of unique chars we need
    int formed = 0;              // number of unique chars currently satisfied

    unordered_map<char, int> windowCounts;
    int left = 0;
    int minLen = INT_MAX;
    int minLeft = 0;

    for (int right = 0; right < s.size(); right++) {
        char c = s[right];
        windowCounts[c]++;

        // Check if the current character's frequency matches what we need
        if (need.count(c) && windowCounts[c] == need[c]) {
            formed++;
        }

        // Try to contract the window
        while (formed == required) {
            // Update answer
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                minLeft = left;
            }

            // Remove leftmost character from window
            char leftChar = s[left];
            windowCounts[leftChar]--;
            if (need.count(leftChar) && windowCounts[leftChar] < need[leftChar]) {
                formed--;
            }
            left++;
        }
    }

    return (minLen == INT_MAX) ? "" : s.substr(minLeft, minLen);
}

int main() {
    cout << minWindow("ADOBECODEBANC", "ABC") << endl;  // "BANC"
    return 0;
}
```

The `formed == required` check is the clever part. Instead of checking all characters every time (which would be O(|t|) per step), we maintain a counter that tracks how many character requirements are currently met. This keeps the inner work O(1).

Let's trace the key moments:

```
Window grows: "A" → "AD" → "ADO" → "ADOB" → "ADOBE" → "ADOBEC"
  At "ADOBEC": formed=3 (A:1, B:1, C:1) — all satisfied!
  Record "ADOBEC" (len=6). Shrink:
    Remove 'A': formed drops to 2. Stop shrinking.

Window grows: "DOBEC" → "DOBECO" → "DOBECOD" → "DOBECODE" → "DOBECODEB"
  → "DOBECODEBA"
  At "DOBECODEBA": formed=3 again.
  Record length=10 (worse). Shrink:
    Remove 'D': still formed=3. Record "OBECODEBA" (len=9, worse).
    Remove 'O': still formed=3. Record "BECODEBA" (len=8, worse).
    Remove 'B': still formed=3 (we have another B). Record "ECODEBA" (len=7, worse).
    Remove 'E': still formed=3. Record "CODEBA" (len=6, tied).
    Remove 'C': formed drops to 2. Stop.

Window grows: "ODEBAN" → "ODEBANC"
  At "ODEBANC": formed=3.
  Record "ODEBANC" (len=7, worse). Shrink:
    Remove 'O': formed=3. Record "DEBANC" (len=6, tied).
    Remove 'D': formed=3. Record "EBANC" (len=5, better!).
    Remove 'E': formed=3. Record "BANC" (len=4, better!).
    Remove 'B': formed drops to 2. Stop.

Answer: "BANC"
```

---

## The Sliding Window Template

Almost every variable-size sliding window problem follows this skeleton:

```cpp
int left = 0;
// some data structure to track window state

for (int right = 0; right < n; right++) {
    // 1. Add nums[right] (or s[right]) to the window state

    // 2. While the window is invalid / oversized:
    while (/* window condition is violated */) {
        // remove nums[left] from window state
        left++;
    }

    // 3. Update the answer
    // For "longest" problems: answer = max(answer, right - left + 1)
    // For "shortest" problems: update answer inside the while loop
}
```

The only things that change between problems:
- What "window state" means (sum, frequency map, set, etc.)
- What the "condition" is (sum >= target, no duplicates, contains all chars, etc.)
- Whether you're maximizing or minimizing

---

## Practice Problem: Maximum Consecutive Ones III

**Problem:** Given a binary array `nums` and an integer `k`, return the maximum number of consecutive 1's if you can flip at most `k` zeros.

```
Input:  nums = [1,1,1,0,0,0,1,1,1,1,0], k = 2
Output: 6  (flip the two zeros at index 5 and 10: [1,1,1,0,0,1,1,1,1,1,1])
```

Stop and think about this for a moment. How would you model this as a sliding window problem?

...

The key reframing: "Find the longest subarray with at most k zeros."

We don't actually flip anything. We just find a window where the count of zeros is <= k.

```cpp
#include <iostream>
#include <vector>
using namespace std;

int longestOnes(vector<int>& nums, int k) {
    int left = 0;
    int zeroCount = 0;
    int maxLen = 0;

    for (int right = 0; right < nums.size(); right++) {
        if (nums[right] == 0) zeroCount++;

        while (zeroCount > k) {
            if (nums[left] == 0) zeroCount--;
            left++;
        }

        maxLen = max(maxLen, right - left + 1);
    }

    return maxLen;
}

int main() {
    vector<int> nums = {1,1,1,0,0,0,1,1,1,1,0};
    cout << longestOnes(nums, 2) << endl;  // 6
    return 0;
}
```

Clean. 12 lines of logic. O(n) time, O(1) space.

The problem said "flip zeros" but the solution never flips anything. **Reframing the problem** is often the hardest part.

---

## When Sliding Window Doesn't Work

Sliding window requires one critical property: when you expand the window, the "cost" increases (or at least doesn't decrease), and when you contract, it decreases. More formally:

- For sum-based problems: elements must be **positive** (or at least non-negative). If elements can be negative, expanding the window might decrease the sum, which breaks the monotonicity that makes contraction meaningful.
- For count-based problems: adding an element can only increase the count of something.

If elements can be negative and you need the maximum subarray sum, you need **Kadane's algorithm** — not sliding window. Different technique, different day.

---

## Recap: How to Recognize Each Pattern

| Signal | Technique |
|--------|-----------|
| Sorted array + pair search | Two pointers (opposite ends) |
| In-place array modification | Two pointers (slow-fast) |
| Partitioning elements | Two pointers (slow-fast) |
| Contiguous subarray/substring, fixed size | Fixed sliding window |
| Contiguous subarray/substring, optimal size | Variable sliding window |
| "Longest/shortest subarray with property X" | Variable sliding window |
| Positive elements + sum condition | Variable sliding window |
| "Contains all characters of..." | Sliding window + frequency map |

---

## Exercises to Solidify Your Understanding

Try these in order. Each one builds on what you've learned:

1. **Container With Most Water** — Two pointers from both ends. Think about why moving the shorter pointer is always correct.

2. **Sort Colors (Dutch National Flag)** — Three-way partition using two pointers. A beautiful algorithm.

3. **Trapping Rain Water** — Two pointers from both ends. For each position, the water trapped depends on the minimum of the max heights on both sides.

4. **Maximum Average Subarray I** — Fixed-size sliding window. Warm-up.

5. **Longest Substring with At Most K Distinct Characters** — Sliding window + hash map tracking distinct character count.

6. **Permutation in String** — Sliding window with a fixed size equal to the pattern length. Compare frequency maps.

7. **Subarrays with K Different Integers** — Hard. Use the trick: `exactly(k) = atMost(k) - atMost(k-1)`. Two sliding window passes.

For each problem, before coding:
- Identify: is this two pointers or sliding window?
- What's the window state?
- What's the condition for expansion/contraction?
- What's the answer being tracked?

Once you can answer these four questions, the code writes itself.

---

## Final Thought

Two pointers and sliding window are not really two separate techniques. They're the same idea: **maintain a range defined by two indices, and move them intelligently to avoid redundant work.** The difference is just in how and why the pointers move:

- **Two pointers (opposite direction):** Exploit sorted order to prune the search space.
- **Two pointers (same direction):** Process elements in a single pass with two different roles (reader/writer).
- **Sliding window:** Maintain a contiguous range that satisfies some property, expanding and contracting to find the optimum.

Master the pattern, and you'll recognize it in problems that never mention "two pointers" or "sliding window" by name. That's the real skill — not memorizing solutions, but seeing the structure underneath the problem.
