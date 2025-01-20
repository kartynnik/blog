+++
title = 'Fair Sampling'
date = 2025-01-16T04:46:37+01:00
draft = false
summary = "Reservoir Sampling redux"
description = "Reservoir Sampling redux"
toc = true
readTime = true
autonumber = false
math = true
tags = ["algorithms", "reservoir sampling", "streaming"]
showTags = false
hideBackToTop = false
+++

## Introduction
Let's assume you have a collection of photos and each photo has a key attribute,
the photographer. Now suppose you want to pick $N$ photos of this collection
for an exhibition, optimizing for the better representation of authors.
Specifically, you want to avoid the following "unfair" situation: one
photographer has $k$ photos exhibited and at least one unexhibited, whereas
another photographer has at least $k + 2$ photos at the exhibition. Taking the
unexhibited photo of the former photographer instead of one of the $k + 2$ photos
of the latter will decrease the absolute difference between the numbers of photos
taken from the two photographers, which in our understanding is going to be fairer.

Call the *signature* of a sample the mapping from keys to their multiplicities
in that sample. I.e. the sample $\{(Peter, photo_1),$ $(Peter, photo_2),$
$(Peter, photo_3),$ $(Steven, photo_4)\}$ has a signature $\{(Peter, 3),$
$(Steven, 1)\}$. If (and only if) Steven has other photos, this is considered
unfair, because one can choose 4 photos with a signature $\{(Peter, 2),$ $(Steven,
2)\}$. (Note that if Steven doesn't have any more photos to choose from, we have
to accept $\{(Peter, 3), (Steven, 1)\}$ as fair.) Observe that the fairness of a
sample can be determined solely by looking at its signature and the signature of
the "general population", i.e. the collection of all the photos in existence.

Not only do we want to produce a *fair* sample according to the above definition,
but we also want to pick it uniformly at random among all the fair samples. It
means every fair sample should have the same probability of being picked.

The following algorithm that has access to the entire dataset produces fair
$N$-samples for our purposes:
```
Repeat N times:
  Sample without replacement a (key, value) pair uniformly
  among those pairs whose keys are least represented
  in the subsample collected so far.
```

(*Without replacement* means that once an item has been picked, it's removed
from consideration in further choices. A sample without replacement is a proper
set, whereas sampling with replacement produces multisets.)

What follows is a lengthier alternative to the definition through that algorithm.

The signature of the subset returned by the algorithm should be *uniformly random*
among the signatures of all the size-$N$ "fair" subsets (i.e., all such signatures
should have equal probability). A subset is called "fair" if the following
condition holds. Assume $M$ is the largest multiplicity of a key in the subset.
Then for all the keys that have multiplicity less than $M$ in the whole set they
should have all their key-values in the picked subset; and all the keys that have
multiplicity at least $M$ in the whole set should be present not less than
$(M - 1)$ times in the subset.

Additionally, the returned subset should be uniformly random among all the "fair"
subsets with the same signature.

As a specific example, if the amount of unique keys seen so far is at least $N$,
only subsets with no duplicate keys are considered.

## Complication
The above algorithm is trivial to implement when the entire dataset is available.
But what if we want to maintain a fair sample of a streaming dataset in an online
fashion, i.e. be able to give a fair sample of the elements encountered so far
at any point in a potentially unlimited stream of key-value pairs?

For that, we'll revisit the classical *reservoir sampling* algorithm.

## Reservoir sampling
The reservoir sampling ["Algorithm R"](https://en.wikipedia.org/wiki/Reservoir_sampling#Simple:_Algorithm_R)
represents a streaming alternative to the following algorithm:
```
Repeat K times:
  Sample without replacement a value from the given set.
```

To arrive at reservoir sampling, one could start with
[Fisher--Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle#The_modern_algorithm).
It is easy to see that sampling $K$ times without replacement from a set represented
as a list is equivalent to shuffling the entire list and taking the first $K$ elements.

A [possible (albeit unpopular) variant of Fisher--Yates shuffle](https://stackoverflow.com/questions/68064254/correctness-of-fisher-yates-shuffle-executed-backward)
looks like this (assuming a zero-indexed array `a`):
```
For i from 1 to n - 1:
  j <- uniformly random integer on the interval [0, i]
  Swap a[j] and a[i]
```

Now if we don't know the entire array in advance and its elements arrive one by
one, we can maintain the first $K$ elements of `a` in `r` and consider the newly
added element to be `a[i]`. If `j < K`, we set `r[j]` to `a[i]`; otherwise,
we discard `a[i]` because in future shuffles it will never occur among the first
$K$ elements. This is what we get (in Python):
```python
class ReservoirSampler:
    def __init__(self, k: int) -> None:
        self._k = k
        self._total_items = 0
        self._reservoir = []

    def sample(self) -> list[Any]:
        return self._reservoir

    def add(self, item: Any) -> Any:
        """Adds an item to the reservoir.

        Returns the item that was removed from consideration, if any.
        """
        self._total_items += 1
        if len(self._reservoir) < self._k:
            self._reservoir.append(item)

        j = random.randrange(self._total_items)
        if j >= self._k:
            return item

        kicked = self._reservoir[j]
        self._reservoir[j] = item
        if self._total_items > self.k:
            return kicked
        else:
            self._reservoir[-1] = kicked
            return None

```
It can be shown that if we're interested in the sample irrespective of the order,
we can avoid shuffling the first $K$ elements among themselves --- i.e. add a
`return` after `append`. However, this property of uniformly shuffled `sample`
outputs will come in handy in a bit.

A similar (but different) derivation of Algorithm R from a variant of
Fisher--Yates shuffle is presented [here](https://en.wikipedia.org/wiki/Reservoir_sampling#Relation_to_Fisher%E2%80%93Yates_shuffle).

## Dynamic reservoir
The Fisher--Yates shuffle intuition allows us to go further in the amount of
unknowns. Imagine that we don't know $K$ in advance. We can keep accumulating
items until we learn what the value of $K$ is, at which point, if $K$ is smaller
than the reservoir accumulated so far, we just leave out the excessive items.
The uniform shuffling of the entire reservoir gives the guarantee that the
remaining sample is uniformly random. We can even support decreasing
$K$ multiple times!
```python
T = typing.TypeVar('T')

class AdaptiveReservoirSampler:
    def __init__(
        self,
        random_uint_less_than: Callable[[int], int] = random.randrange
    ) -> None:
        """Constructs the sampler.

        Args:
          random_uint_less_than: A callable that produces
              a uniformly random integer in range [0, N)
              when called with the parameter N.
        """
        self._random_uint_less_than = random_uint_less_than

        self._reservoir_mode = False  # Unbounded K.
        self._reservoir = []
        self._total_items = []

    def sample(self) -> list[T]:
        return self._reservoir

    def limited(self) -> bool:
        return self._reservoir_mode

    def limit(self) -> None:
        self._reservoir_mode = True

    def __len__(self) -> int:
        return len(self._reservoir)

    def __iter__(self) -> Iterable[T]:
        return iter(self._reservoir)

    def add(self, item: T) -> T | None:
        """Adds an item to the reservoir.

        Returns the item that was removed from consideration, if any.
        """
        self._total_items += 1
        # `not self._reservoir_mode` effectively means infinite K
        if not self._reservoir_mode:
            self._reservoir.append(item)

        j = self.random_uint_less_than(self._total_items)
        if self._reservoir_mode and j >= len(self._reservoir):
            return item

        kicked = self._reservoir[j]
        self._reservoir[j] = item
        if self._reservoir_mode:
            return kicked
        else:
            self._reservoir[-1] = kicked
            return None

    def remove_one(self) -> T:
        return self._reservoir.pop()
```

An alternative is to forgo the shuffling of the reservoir, but
when cutting it down by one, to perform a deferred exchange of the last element
subject to removal with a random element of the reservoir in order to maintain
uniformity. If decreases are rare, this can save some calls to the randomizer:
```python
    def add(self, item: T) -> T | None:
        """Adds an item to the reservoir.

        Returns the item that was removed from consideration, if any.
        """
        self._total_items += 1
        if not self._reservoir_mode:
            self._reservoir.append(item)
            return None

        j = self.random_uint_less_than(self._total_items)
        if self._reservoir_mode and j >= len(self._reservoir):
            return item

        kicked = self._reservoir[j]
        self._reservoir[j] = item
        return kicked

    def remove_one(self) -> T:
        j = self.random_uint_less_than(len(self._reservoir))
        kicked = self._reservoir[j]
        self._reservoir[j] = self._reservoir[-1]
        self._reservoir.pop()
        return kicked
```

## Putting it to use
Let's return to our problem of fair sampling. First of all, observe that once it
has been determined how many values for a certain key are chosen, it comes down
to sampling that many values from the overall set of values with this key. In
other words, for every key can keep an adaptive reservoir sampler over its keys
(unfortunately, that means we have to store all the encountered keys).
```python
K = typing.TypeVar('K')
V = typing.TypeVar('V')

class FairSampler:
    def __init__(
        self,
        size_limit: int,
        random_uint_less_than: Callable[[int], int] = random.randrange
    ) -> None:
        self._size_limit = size_limit
        self._random_uint_less_than = random_uint_less_than

        self._size = 0
        self._key_to_values = collections.defaultdict(
            lambda: AdaptiveReservoirSampler(
                self._random_uint_less_than))
        # To be continued...
```

As in the reservoir sampling case, we'll simply accumulate all the observed items
until their number exceeds `size_limit`. This is where the interesting part comes.
Assume we add `(key, value)` and the multiplicity of `key` becomes $m$. If it is
less than the maximal multiplicity $M$ of the keys in the set, we have to accept
this pair (due to fairness constraints); but we also have to reduce the number of
the keys with multiplicity $M$ in our signature in order to fit the size limit.
To address that in a uniform fashion, we'll maintain a reservoir of keys with
multiplicity $M$. Since with time this multiplicity can be depleted, we'll store
a list of keys for every given multiplicity $m$ --- as an adaptive reservoir that
can become limited as it becomes the maximal multiplicity overall.
```python
        # i-th level is an `AdaptiveReservoirSampler` of the keys
        # occurring exactly (i + 1) times in the current sample.
        self._levels = []

    def add(self, key: K, value: V) -> None:
        values = self._key_to_values[key]
        if values is None:
            # To save memory, we store `None`
            # instead of empty limited reservoirs.
            return
        values.add(value)
        if values.limited():
            # This was already a limited reservoir,
            # the signature is preserved.
            return

        multiplicity = len(values)
        if multiplicity > len(self._levels):
            # Create a new level for the new largest multiplicity.
            self._levels.append(
                AdaptiveReservoirSampler(self._random_uint_less_than))
            assert multiplicity == len(self._levels)

        maybe_kicked_key = self._levels[multiplicity - 1].add(key)

        last_level = self._levels[-1]
        if self._size == self._size_limit:
            # Too many items in the sample,
            # have to kick one value of a top-level key.
            if multiplicity == len(self._levels) and (
                    last_level.limited()):
                # When we were adding a new key
                # to the last level, some key got kicked.
                kicked_key = maybe_kicked_key
            else:
                # We have too many keys in the last level now,
                # time to shrink and limit it.
                last_level.limit()
                kicked_key = last_level.remove_one()

            if not last_level:
                self._levels.pop()

            self._kick_one_value(kicked_key)
        else:
            # Just keep adding without limitations.
            self._size += 1

    def _kick_one_value(self, key: K) -> None:
        values = self._key_to_values[key]
        values.limit()
        values.remove_one()
        if not values:
            # Instead of storing an empty limited reservoir.
            self._key_to_values[key] = None
```
To return a sample, we trivially combine the reservoirs of the keys at level 0,
which is guaranteed to contain all the keys with at least one value:
```python
    def get_sample(self) -> Iterable[tuple[K, V]]:
        return [(key, value)
                for key in self._levels[0]
                for value in self._key_to_values[key]]
```

## Testing uniformity of randomness
Suppose we want to write a unit test for the above code. A couple of questions
immediately arise:

1. How to deal with the fact that the algorithm's outcomes are random?
2. How to check the claim that the outcomes are uniformly distributed?

Here's where that `random_uint_less_than` argument for the fair sampler comes in.
Instead of producing a single random value, we may "split the multiverse" across
all the possible values, branching out in a tree of the possibilities of history
unfolding. The second call to `random_uint_less_than` will be similarly branched,
etc., and the results from the algorithm finishes will be available in the leaves
of this tree of histories. Re-running the algorithm for each path in this tree
and picking the results at the leaves gives you the entire
picture of the probability distribution: The probability of a leaf is easily
computed from the probabilities of the branches on the path to it, which are one
over the branching fan-out individually and are independent so can be multiplied.

```python
class MultiverseRandomnessSource(object):
    """Allows to explore all the possibilities of dice rolls.

    For an algorithm that depends on a one-argument
    `random.randrange()`-style source of randomness
    and becomes deterministic once its outcomes are fixed,
    allows to explore a tree of all the possibile of tracks
    of the algorithm by traversing a tree of states
    that the algorithm enters prior to each roll.
    A state's children in the tree represent each
    of the possible roll outcomes.
    (Implementation detail: The tree itself is not stored,
    only the current path.)
    """

    class _Node(object):
        """Represents a point of execution.

        (At either a dice roll or termination.)

        Stores the dice value limit as well as the last roll result
        (to either continue exploring this subtree or switch to the
        next one onwards) unless it is a termination state.
        """

        def __init__(self, parent: Optional['_Node'] = None):
            # The non-inclusive upper limit for the
            # random number requested from here.
            # `None` denotes a leaf (always appears
            # only at the end of the path).
            self.limit = None

            # The "random" value we are going
            # to give back next time if asked.
            self.next_answer = 0

            # The reciprocal of the probability
            # of reaching this branching point.
            self.probability_reciprocal = 1
            if parent is not None:
                self.probability_reciprocal = (
                    parent.probability_reciprocal * parent.limit)

        def is_leaf(self) -> bool:
            return self.limit is None

    def __init__(self) -> None:
        # The current path along the tree
        # of execution path possibilities.
        # Its prefix down to `self._current_depth`
        # is the already passed subpath.
        self._path = [self._Node()]  # The root that is also a leaf.
        # The current depth of descent into the tree.
        self._current_depth = 0

    def random_uint_less_than(self, limit: int) -> int:
        if limit < 1:
            raise ValueError('Empty randomization range')

        assert self._current_depth < len(self._path)

        parent_state = self._path[self._current_depth]
        if parent_state.is_leaf():
            # The leaf can only appear at the end of the path.
            assert self._current_depth + 1 == len(self._path)

            # Branch the leaf.
            parent_state.limit = limit

            # Add a new child leaf.
            leaf = self._Node(parent=parent_state)
            self._path.append(leaf)

        elif parent_state.limit != limit:
            # In a different run, the algorithm
            # rolled a dice with a different limit.
            raise ValueError('Another non-determinism source '
                             'detected: different range limit')

        returned_value = parent_state.next_answer
        self._current_depth += 1

        return returned_value

    def rewind(self) -> None:
        if self._current_depth + 1 < len(self._path):
            # In a different run, the algorithm didn't stop here.
            raise ValueError('Another non-determinism source '
                             'detected: did not stop previously')

        assert len(self._path) >= 1
        assert self._path[-1].is_leaf()

        # Exit from the leaf.
        self._path.pop()

        # Exit from the subtrees that we have fully explored.
        while self._path:
            last_branch = self._path[-1]
            assert not last_branch.is_leaf()

            last_branch.next_answer += 1
            if last_branch.next_answer == last_branch.limit:
                self._path.pop()
            else:
                break

        if self._path:
            # Denotes the first leaf of the deepest remaining branch.
            leaf = self._Node(parent=self._path[-1])
            self._path.append(leaf)

        # Be ready to start over.
        self._current_depth = 0

    def explore(
        self, algorithm: Callable[[], T]
    ) -> Iterable[tuple[T, fractions.Fraction]:
        """Explores all the algorithm run outcomes.

        Provides their probabilities.

        Args:
            algorithm: A callable employing solely
                this `MultiverseRandomnessSource`
                as its exclusive source of randomness.
        Yields:
            (result, probability):
                One per each leaf in the tree of nondeterministic
                computation paths. `result` is the result
                of running the algorithm along the given path.
                `probability` is a `fractions.Fraction` instance
                representing the probability of reaching this leaf.
        """
        # While there's still something to explore...
        while self._path:
            result = algorithm()

            leaf = self._path[-1]
            assert leaf.is_leaf()

            probability = fractions.Fraction(
                1, leaf.probability_reciprocal)

            yield result, probability

            self.rewind()
```
This is how this would be used in a unit test:
```python
multiverse = MultiverseRandomnessSource()

def algorithm() -> Iterable[tuple[K, V]]:
    sampler = FairSampler(
        size_limit,
        random_uint_less_than=multiverse.random_uint_less_than)
    for key, value in sequence:
        sampler.add(key, value)
    return sampler.get_sample()

all_possible_outcomes_with_probabilities = multiverse.explore(
    algorithm)
# Check the properties of the outcomes, in particular,
# that the outcomes follow the definition of fairness and that
# total probability of a given outcome doesn't depend on the outcome.
```
## Open question
Is it possible to avoid storing all the keys?
