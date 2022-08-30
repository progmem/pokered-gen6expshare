# Pokémon Red/Blue Gen VI-style EXP Share

I have a love-hate relationship with Pokémon games. I grew up with the original games, having played Blue and Gold to as much completion as I could have for someone who didn't know anyone else with a Gameboy at the time. I fell off during the GBA games, tried to get back into it with Pearl, and just haven't really been back into the series as much, at least, until the Let's Go games came out.

The Let's Go series of Pokémon games are effectively a remake of Pokémon Red/Blue, only you start with a Pikachu or Eevee. Instead of getting into random battles and beating wild Pokémon to submission, you run into a model on the world and lob a Pokéball at it, similar to Pokémon Go. Catching wild Pokémon became much easier. Most importantly, Pokémon now leveled up similar to the modern games.

In the more recent titles, the ability to share experience is either a toggleable option, or forced-on by default. Most importantly, experience is handed out in what is effectively a more simpler formula. I don't know the exact formula, but it's _basically_:

* A Pokémon gets 100% of the experience it would if it participated in battle.
* Otherwise, they get 50% of the experience they would've if they participated.

Recently, I saw this [lovely Gameboy Color hack for Pokémon Red and Blue](https://www.romhacking.net/hacks/1385/), which colorizes and optionally replaces the spritework in battles with those from Gold and Silver. This made me want to get back into playing Pokémon, so I booted up Red, got my Bulbasaur, caught a couple Pokémon and...decided this was too slow. I couldn't go back.

So, let's go forward!

## Usage

If you just want to use these patches, download your respective `.ips` file for your version of the game (Red or Blue). Apply the patch and start playing!

If you want to use this code in your own ROM hack, the assumption going forward is that your hack is based on the [Pokémon disassembly over here](https://github.com/pret/pokered/). Assuming you're starting from scratch, use the `core.asm` and `experience.asm` files here in place of the original files, or extract the necessary bits for your own needs.

## How Pokémon Get Big & Strong

Normally, all Pokémon that participate in battle receive a divided amount of experience based on the number of participants any time an enemy Pokémon faints. A single participant receives 100% of the experience. Two participants receive 50% each. The more participants, the less experience per Pokémon and the longer the grind is.

However, this mechanic changes if the Exp. All item is in the player's bag. The Exp. All item changes the formula in order to spread experience across the entire party. This is _supposed_ to work like so:

* First, divide the experience into two separate pools.
* The first pool should be granted across all Pokémon that participated in battle, just like before.
* The second pool is divided evenly across all Pokémon in the party. Pokémon that participated in battle will receive some experience from both pools.

However, this [doesn't quite work right](https://bulbapedia.bulbagarden.net/wiki/Exp._Share#Generation_I). The second pool is actually sized to the amount that a single battle participant receives, and it's _that_ amount that's divided. Additionally, any experience that would go to a fainted Pokémon is just outright lost.

In short, Exp. All is a broken and bad item, but we can fix it. Even better, we can make it stay in effect the entire time. We can do so by [hacking on the ROM](https://github.com/pret/pokered/). I'll be referring to this project's code going forward, and calling out source files from this project.

### Bag Check

The `FaintEnemyPokémon` routine in `engine/battle/core.asm` does a lot of stuff. One thing it does is to perform a check in the player's inventory to see if the `EXP_ALL` item exists. If it does, the enemy stats used to calculate experience are all halved (`.halveExpDataLoop`). Then, after experience is granted to all participants (`.giveExpToMonsThatFought`) by jumping to `GainExperience`, a second portion of code is ran if the `EXP_ALL` item exists. The memory holding all participating Pokémon (`wPartyGainExpFlags`) is updated to appear as if all Pokémon participated, and `GainExperience` is jumped to a second time.

### The Great Divide

The `GainExperience` routine in `engine/battle/experience.asm` gives each Pokémon some experience as long as the battle didn't occur during a Link Battle. As part of this routine, `DivideExpDataByNumMonsGainingExp` is called to divide the amount of experience to grant by the number of Pokémon flagged in `wPartyGainExpFlags`. Participants are counted via a loop (`.countSetBitsLoop`), and the division happens in its own loop (`.divideLoop`). This gives us the amount of experience each Pokémon should receive.

## The Plan

At this point, we understand a fair bit on how Pokémon works under the hood. A byte, `wPartyGainExpFlags`, contains the flags of all Pokémon that participated in battle, and this is referenced by `GainExperience` to decide who should get how much experience. So, let's revisit our goals:

* Give every battle participant 100% experience, _not divided_. If there's two participants, each participant receives 100% experience.
* Give every other Pokémon (non-participants) 50% experience, again _not divided_. If there's four non-participating party members, they each receive their own 50% experience.

So, we need a way to keep track of who participated and who didn't. Thankfully, we can use some bitmath to do this.

### Just a Little Bit

For those who have _never played Pokémon_, you can have a party of six (6) Pokémon at your disposal. If every single party member participates in one battle (by swapping them into battle), `wPartyGainExpFlags` would be set to `$3f`. `wPartyGainExpFlags` is a byte, and therefore has a max value of `$ff`. Pokémon does not use the two most significant bits, so let's make use of them ourselves. We'll use `$80` to decide _whether or not to divide_. Specifically, we'll use the division in `DivideExpDataByNumMonsGainingExp`. and completely remove the division that happens as part of `.halveExpDataLoop`. We can then rewrite a fair chunk of `DivideExpDataByNumMonsGainingExp` to do the following:

* Check if bit 7 `$80` is set in `wPartyGainExpFlags`. If it's not, `ret`urn early to skip the divide to ultimately give every Pokémon set in `wPartyGainExpFlags` 100% experience.
* If we're still here, we take `wPartyGainExpFlags`, `xor $80` to unset the bit we're using, and store the updated value back in `wPartyGainExpFlags`. This is important, since otherwise we end up with an edge case that the original Pokémon code doesn't know how to deal with.
* We'll retrieve the enemy stats and perform `.divideLoop` as the original code does with one change: we load our `hDivisor` with the value `$2` as we're dividing in half no matter what here.
### The Mask feat. Son of the Mask

Back in `FaintEnemyPokémon`, we can completely remove all code starting from our `EXP_ALL` check. We no longer need to check for the item and `push af` to preserve this check for later, so we can do the following instead:

* In our replacement code (`.giveFullExperience`) we call the same code as what was originally called to grant 100% experience (`.giveExpToMonsThatFought`). We will not need to `pop af` off the stack, and we won't be returning right away.
* Next, we'll build a mask `wPartyCount` long (`.buildPartyMask` and `.partyMaskLoop`). This is done to create a precise mask we can then `xor` in order to avoid creating edge cases.
* Next, we'll check if we should give half experience (`.checkForHalfExperience`). We use the mask we built by `xor`ing it against `wPartyGainExpFlags`, giving us the list of Pokémon who did not participate. If, after `xor`ing we're left with 0, no Pokémon should be given 50% experience and we can `.skipHalfExperience`.
* Finally, we `xor $80` to set the bit we need to check for in `DivideExpDataByNumMonsGainingExp`. When we call `GainExperience` and perform the divide this bit will be used to kick off the divide.

## Lessons Learned

Pokémon is a _very optimized game_, and because of this a lot of assumptions are made, and quite a few bugs occur as a result of these optimizations. The original hack I did for this was done in ~30 minutes, but in doing so I had opened up a world of edge cases in the ROM. The remaining day that was spent on this was a number of false starts in fixing these issues, and ultimately we settled on playing by the rules the developers at Game Freak established to avoid, say, granting one extra party member an additional amount of experience, or crashing the game outright (though I determined this to be the accidental removal of a `ret`).

The end result? The Magikarp you can buy for 500 Pokedollars is now _totally worth every penny_. By the time I got my first Nugget, I had a Gyarados, and with a minimal amount of purposeful grinding. Now, the games I grew up with respect what little time I have these days, and that makes me happy.