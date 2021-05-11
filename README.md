# arknights-riic-planner
An Arknights planner that optimizes RIIC production in addition to stage farming.

This planner can accessed as a webpage via the following [Google Colab link](https://colab.research.google.com/drive/1lHwJDG7WCAr3KMlxY-HLyD8-yG3boazq?usp=sharing).

## Introduction

Greetings doctor!

I'm sure by now you've probably seen your fair share of farming planners for Arknights, and possibly used their analysis to help you decide which stages are most efficient to farm for you. At its core, this planner is much the same; it uses the same linear-programming techniques to figure out (i) which stages to farm and (ii) how much sanity each material is worth. The main advancement in this planner that it <em>factors in your RIIC</em>, which is responsible for a large part of your LMD/exp production. If you've ever been curious about what type of base to run (2 trading posts and 4 factories, or 3 trading posts and 3 factories?), what you should be boosting with your drones (trading posts or factories?), or even how to assign a sanity value to pure gold, then I hope this planner will be useful to you.

In the making of this planner, I have tried my best to account for a variety of game mechanics related to the way you earn materials. As a result, this planner has a long list of fields meant to better capture your unique situation. I understand that it is a hassle to compute these values and fill them in, so this planner comes with default values that I recommend sticking with initially. There are game mechanics that this planner does not currently capture, sometimes due to modeling difficulty (e.g. the recruitment permit <-> green certs cycle), sometimes due to lack of good data (e.g. how the friend credits shop stocks items), sometimes both. I look forward to future work that addresses these concerns!

Cheers,
Luzark

## Planner Comparison

Here are my notes on different Arknights planners to help you decide which one fits your use case. I am aware of the following planners and will abbreviate them as follows:
- [`LUZ`](https://colab.research.google.com/drive/11NvYlUdAmTAUGpkYWEdSF_j2GfkQgS25?usp=sharing): This planner.
- [`AP`](https://penguin-stats.io/planner): Arkplanner, written by 🦀 and 根派. The backend engine for [ArkOneGraph](arkonegraph.herokuapp.com).
- [`GP`](https://gamepress.gg/arknights/tools/arknights-operator-planner-beta): Gamepress' planner, managed by EmmaNielson, Alyeska, and NorseFTX.
- [`MN`](https://www.reddit.com/r/arknights/comments/i0y1x8/arknights_priority_planner_update_4_now_with/): Written by [u/MathigNihilcehk](https://www.reddit.com/user/MathigNihilcehk/).
- [`MOE`](https://docs.google.com/spreadsheets/d/12X0uBQaN7MuuMWWDTiUjIni_MOP015GnulggmBJgBaQ/edit): Written by [u/elmoe0715](https://www.reddit.com/user/elmoe0715).

Obviously, the table entries below are based on my limited interactions with these planners. If unsure, I've annotated the entry with a question mark. If you have a correction to this table, please feel free to reach out to me (see Contact Info section). This table was written on August 5th, 2020, and is therefore unaware of planner updates since then.

| **Planner** | `LUZ` | `AP` | `GP` | `MN` | `MOE` |
| :-: | :-: | :-: | :-: | :-: | :-: |
| **User Interface** | Google Colab | Website | Website | Microsoft Excel | Google Sheets |
| **Objective Function** | Demand Vector | Demand Vector | Demand Vector | Demand Vector | Supply-Side |
| **Workshop Recipes** | Elite + Chips | Elite | Elite T4/T5 + Chips | Elite | Elite |
| **Workshop Byproducts** | Weighted Byproducts | Yes? | No Byproducts | Yes? | Weighted Byproducts |
| **Skill Summaries** | Yes | No | Yes | Yes | No |
| **Pure Gold Valuation** | Via LP Constraints | Via High Demand | No | No | Via Supply-Side Objective |

## Discussion of Techniques

### Problem Statement

The primary goal of planners is to solve the following problem:
> **Key Problem.** What's the fastest way to farm this set of materials?

The typical output of such a planner is the number of times to run each stage; let us refer to this as a "farming plan". A valid farming plan is one that produces the requested set of materials, and the optimal farming plan is the one that uses the least sanity among all valid farming plans.

Of course, there is randomness in what a stage will drop from run to run. For simplicity, we (and other planners) use the expected drop assumption:
> **Expected Drop Assumption.** Stages deterministically drop their expected materials. For example, PR-A-1 drops half of a defense chip, half of a medic chip, and 216 LMD.

This obviously is not true, but it greatly simplifies the math. Without this assumption, the optimal farming plan would be *adaptive*; based on what dropped from a stage, you would determine which stage to farm next.

### Linear Programming 101

Fortunately, our problem fits quite cleanly into the [Linear Programming](https://en.wikipedia.org/wiki/Linear_programming) (LP) framework. The gist of LP theory is that if your optimization problem can be written as a linear objective subject to linear constraints, then it can be efficiently solved. We can confirm that our key problem meets these requirements:
- Our problem variables are, for each stage, the number of times to farm that stage.
- Our problem objective is to spend the minimum amount of sanity. The amount of sanity spent is just (sanity cost of 0-1) x (times we run 0-1) + (sanity cost of 0-2) x (times we run 0-2) + ... This objective is indeed linear in our problem variables.
- Our constraints are that our farming plan must be valid. Let's concentrate on a single material we care about, e.g. LMD. If our farming plan is valid, then (LMD reward of 0-1) x (times we run 0-1) + (LMD reward of 0-2) x (times we run 0-2) + ... >= our LMD goal. This inequality is also linear in our program variables. In fact, we can write down one such inequality for every other material that we care about. If we satisfy all these linear inequalities, then our farming plan is valid.
- As a minor detail, we can't farm stages a negative number of times, so we also need to assert that (times we run 0-1) >= 0, etc. These are also linear inequalities.

Since our problem can be formulated as an LP, it means we can find the optimal farming plan by using an LP solver. However, there's a tiny catch: the number of times we are told to farm each stage might be fractional:

> **Fractional Farming Assumption.** The planner is allowed to run a stage a fractional amount. For example, running CE-5 half a time produces 3750 LMD at the cost of 15 sanity.

Some solvers might avoid this assumption by using a Mixed-Integer Program (MIP) instead, which allow you to ban fractional variable values. Unfortunately, MIPs are [harder to solve](https://en.wikipedia.org/wiki/NP-completeness), so we'll stick to LPs here for efficiency reasons.

### LP Duality and Material Sanity Values

At this point, some of you might be wondering how planners produce sanity values for materials, and what these values mean. It turns out that the theory of LPs informs us that every LP formulation has a [dual LP](https://en.wikipedia.org/wiki/Dual_linear_program) that we can work out by following a bunch of rules that convert our LP to its dual LP (it's called a dual because if we apply the rules again, we wind up with our original LP). There are a number of properties connecting these two linear programs, which we will discuss in this section.

In our case, when we apply the rules to transform our "primal" LP above into its dual, we get the following optimization problem:
- Our problem variables are for each material, its sanity value.
- Our objective is to maximize (demand for diketon) x (sanity value of diketon) + (demand for ester) x (sanity value of ester) + ...
- Our constraints are that, for each stage, the sanity values of the resulting materials cannot exceed the sanity cost of the stage.
- As a minor detail, the sanity value of a material cannot be negative.

In short, sanity values are derived by solving the dual LP problem above (remember, solving LPs is very efficient). The dual LP is deeply connected to the original LP; here's a list of well-known properties and how they translate to our setting when we solve for the optimal farming plan and optimal sanity values:
- **Strong Duality.** The sanity cost of our farming plan is equal to the sanity value of the demanded materials.
- **Complementary Slackness.** The optimal farming plan only uses stages that don't lose any sanity value. A material can only have zero sanity value if the optimal farming plan produces strictly too much of it.

### New Origin Resource: the Day

In our explanation so far, sanity is the resource from which all other resources are derived. Unfortunately, there's not really a direct relationship between sanity and RIIC productivity (ignoring that it can be inefficiently converted into drones). As a result, it's difficult for us to incorporate the RIIC when talking about minimizing sanity usage, since the RIIC doesn't consume sanity normally.

To get around this, the origin resource in this planner is actually the day. Previously, sanity got converted into materials via farming stages. The process is a little more complex now. Days get converted into sanity, factories, and trading posts. Sanity is still converted into materials. Factories are converted into pure gold and battle records. Trading posts and pure gold are converted into LMD.

When we use the above idea to rewrite our LP, several things change. Rather than minimizing sanity spent by the farming plan, we are now minimizing the number of days spent by the farming plan. When we take the dual LP, we actually get material day values rather than material sanity values. However, the fundamental principles remain the same. To make the results of this planner easier to compare against other planners, I have converted the material day values into material sanity values by dividing through by the day-value of one sanity.

### The RIIC

Perhaps the most straightforward way to factor in the RIIC is to note how much yours produces each day in LMD/battle records and treat that as a daily income (indeed, this is the approach taken by the `MN` planner). However, this approach does not let our LP solver freely swap between gold factories, exp factories, and trading posts.

Our approach is to introduce two psuedomaterials: the factory-day and the trading-post-day. The former represents the production of an unboosted factory over one day, while the latter represents the production of an unboosted trading post over one day. Each day, our RIIC could be in one of several configurations. For example, if you are running a 243 (2 trading posts, 4 factories, 3 power plants) base, then it's possible to convert to a 153 or a 333 base.

Without factoring operator bonuses, then, we have several ways to use up one day. For simplicity, let's assume that we receive 240 sanity per day.
- We could convert one farming day into 240 sanity, 1 trading-post-days, and 5 factory-days.
- We could convert one farming day into 240 sanity, 3 trading-post-days, and 3-factory-days.
- We could convert one farming day into 240 sanity, 2 trading-post-days, and 4 factory-days.

As you can see, this lets us exchange trading posts with factories. As a result, when we get a pure gold as a drop for a stage, we could spend some trading post time on converting it to LMD, but the opportunity cost is that we could have made factory time instead and printed battle records (or pure gold).

This planner does the above tradeoff, but also factors in operator bonuses and power plants. In particular, the first factory in your base has much better productivity than the fifth factory because yoru first factory gets its pick of your best operators while the fifth factory is scraping the bottom of the barrel.

## Credits and Contact Information

All names mentioned here are Discord usernames.

I've had many helpful conversations regarding planners with 🍑Moe🍑#2568, who authored the `MOE` planner. Moe has also helped me beta test this planner.

This work also would not have been possible without the aid of the #r-i-i-c club in the Arknights Official Discord Server, which has helped me with several RIIC-related calculations and default values. In particular (and in alphabetical order), I would like to thank Forge#2341, Lance#1186, ScifiGemini0616#3863, and Vinci#4580 for teaching me about RIIC mechanics, helping me generate reasonable default values for this planner, and beta testing this planner.

I'd also like to thank MySteRiouS#0972 for helping me with the monthly log-in mechanics when I was querying #help.

Thanks to Penguin Statistics for collecting invaluable drop data without which this project would not be possible. This project inherits a [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) license from this data.

Thanks to Gamepress for event, operator, and other miscellaneous information.

If you spot a bug or have other feedback, please feel free to contact me as Luzark#8152 on Discord.
