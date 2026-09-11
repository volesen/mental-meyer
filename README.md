# Mental Meyer

[Meyer](https://da.wikipedia.org/wiki/Meyer_%28terningspil%29) is a Danish dice game about bluffing. You roll two dice under a cup, peek, and announce a value. The next player either believes you and rolls again, or lifts the cup to find out whether you lied.

We want to play it peer to peer, without trusting a dealer or another player's dice generator. Like [mental poker](https://people.csail.mit.edu/rivest/pubs/SRA81.pdf) and [coin flipping over a distance](https://en.wikipedia.org/wiki/Coin_flipping#Telecommunications), everyone helps generate the dice, only the roller can inspect them, and a challenge lets everyone check them. Bluffing stays legal.

## Plan of attack

A **commitment** locks in a secret value: publish its salted hash now, reveal the value and salt later. Others can verify the opening without being able to inspect the secret beforehand.

For each roll:

1. **Everyone commits.** Each active player chooses a uniform `share` in `0..35` and a random 32-byte secret `salt`. Publish a commitment. Fix the complete set before anyone reveals anything.
2. **Everyone except the roller opens.** Publish the shares and salts, and verify them against the commitments. The roller keeps their contribution secret.
3. **The roller declares.** Only the roller can calculate the dice. They announce a legal value, which may be a bluff.
4. **The next player decides.** Acceptance retires the roll unopened and starts a fresh roll for the next player. A challenge makes the roller reveal their contribution, so everyone can reconstruct the dice and apply the penalty.

The dice are just the combined contributions:

```text
r  = sum(all shares) mod 36
d1 = 1 + (r mod 6)
d2 = 1 + floor(r / 6)
```

Under the hash-commitment assumptions, one honest player's independent uniform contribution makes the underlying draw uniform over all 36 ordered outcomes. An honest roller's secret contribution hides the dice, except for information they disclose through play.

Cryptography cannot force someone to finish. Invalid required openings or refusal to reveal invoke an agreed forfeit policy, never a free reroll. Selective aborts can still bias which games finish. This sketch uses salted hash commitments in the random-oracle model and covers one private roll per turn.

## The JSON protocol messages

Commitment: players publish the hash of a random 64-bit integer.

```
{ "op": "commit", "hash": 35019804195 }
```

Publish: non-roller players publish their chosen number (receivers verify that the hash matches

```
{ "op": "publish", "num": 1234 }
```

The roller rolls their dice in secret by xor'ing all the published numbers and modulo'ing by 36

The roller then declares what they rolled (may be a bluff) as a number between 0 and 35 inclusive:

```
{ "op": "declare", "roll": 1 }
```

The roll corresponds to these dice rolls by index:

```
[32, 41, 42, 43, 51, 52, 53, 54, 61, 62, 63, 64, 65, 11, 22, 33, 44, 55, 66, 31, 21]
```

The next player then responds with either "lift" or "accept"

```
{ "op": "lift" }
```

```
{ "op": "accept" }
```

In case the next player responds with "lift", the current player must reveal their secret number.

```
{ "op": "lift", "num": 5678 }
```

Then all the other players check by doing the same xor-and-modulo operation. In case the rolling player has cheated, any player can broadcast a "cheat":

```
{ "op": "cheat!" }
```


