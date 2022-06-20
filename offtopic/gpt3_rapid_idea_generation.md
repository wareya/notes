# Rapid idea generation with GPT-3

Rapid idea generation tools* are extremely useful for writers to overcome writer's block, churn through bad ideas to get to good ones, notice holes in their story, get fresh perspective on how their story elements could be used, and entertain themselves while "on the clock" without simply idling the time away.

\* (the established term is "generator", but that's overly vague and doesn't get across what they actually do)

## Why?

Most of the hype around GPT-3 and text generation models is around producing humanlike text, like completions for incomplete stories, or fictional chat logs, or news articles, or scientific explorations. But what if we use it to produce ***machine-like*** text? People use machines to generate random ideas all the time, from fictional place names to random combinations of foods or drinks. But generators are traditionally hard to craft: someone has to sit down and design all the relevant strings or data, and the ways in which they're allowed to interconnect, and how the dice are rolled, etc. If an overpowered text generation model capable of acting like a human could also act like a machine, then perhaps we could shed the need for that design effort, and create arbitrary generators of arbitrary subject matters just by feeding it a few examples.

I tried this with minimal GPT-2 models. It didn't work. They always got stuck in cycles or started spewing random garbage, even if they were fine-tuned.

But with GPT-3, ***it just works***. There are some catches: it really likes to produce certain words at least a few times for certain prompts, and for complex prompts you need to crank the temperature etc. to absurd levels, and for plot/story ideas it ***really*** likes making the protagonist a young woman (a strange form of AI bias--it's probably learning from other plot generators), and for cosmology/origin stories it ***really*** likes talking about "Architects" and "One".

But ***it just works***. ***Without*** fine-tuning. We have unlimited generators--a Generator for Rapid Idea Generation Generators--on our hands.

I don't think I'm the first person to think of this. In fact, I *know* I'm not. But I don't see anyone talking about it, so I'm pretty sure it's an under-explored idea, and I want the idea to spread. Maybe people don't realize just how ridiculously easy it is to make GPT-3 work this way. I repeat: it ***just works***.

Here are some examples. All text past the first highlight is AI-original; it removes the highlighting on continuations for some reason.

I hope this demonstrates to someone how much potential this has and that it gets looked at properly as a viable application of this technology.

## Example: Magical Weapon Generator

Prompt:

```
Magical Weapon Generator

Ideas for magical weapons, from the mundane to the blessed and demonic, from guns to swords and everything in between. Include science fiction items.

(The temperature and penalty parameters in many of these examples are non-default. YMMV.)

Weapons (20):

1: Earthwend's Flameaxe (axe)
2: GLADIUS (railgun for mechas)
3: The Everlight (holy sword)
4:
```

Output:

![image](https://user-images.githubusercontent.com/585488/174666573-a15ddf86-f4cf-41e9-9a99-b0cf901b0cac.png)

## Example: Story Outline Generator

Prompt:

```
Story Outline Generator

Generates outlines for fantasy and scifi stories. Outlines for contemporary stories might contain supernatural elements.

Stories (10):

1: Protagonist is transported to another world while walking home from school. He meets a witch. She brings him to magic school. He gets involved with preventing a plot to assassinate the school's head warlock. After meeting a lot of other students, and his witch friend getting humiliated in magic classes, he uses his Earth knowledge to stop the assassination. He's heralded as a hero.
2: Protagonist wakes up with no memory of who she is. She finds horns on her head. She wanders the village she finds herself in until a tailor's family picks her up. She gets involved with the local ghost-hunting club, which is just for play, but they find a *real* ghost in the clock tower. After she learns that she's a demon, and can use magic, they go back to the clock tower and "send" the ghost to the next world. Everyone lived happily ever after.
3:
```

Output:

![image](https://user-images.githubusercontent.com/585488/174667202-42bad10c-119c-4a73-b96d-3a57f65fb77a.png)

## Example: Fantasy Race Generator

Prompt:

```
Fantasy Race Generator

Generates unusual fantasy races or strange remixes of established ones.

Races (10):

1: The Devils are a sapient race of scaley humanoids that live underground and have very complex internal politics with scandals wrapped inside of scandals. They rarely surface, but when they do, they cause a lot of trouble. This has earned them the ire of other races.

2: The Dwardi are a race of short, stocky, humanlike earth-lovers that live in cold climates and love desolate landscapes. They have a history of living in the mountains, and have a culture that prizes mushroom cultivation and clay-eating. In modern times, they have become industrious traders, connecting the many nations of the world together by railcar.

3:
```

Output:

![image](https://user-images.githubusercontent.com/585488/174667942-1eeb8cee-31af-458f-905b-105d1c728fe8.png)

## Example: Fantasy Cosmology Generator

Prompt:

```
Fantasy Cosmology Generator

Generates origin stories, cosmologies, magic systems, etc. for fantasy and scifi stories.

Cosmologies (10):

1: Everything on which the light shines has a shadow. Both physical and spiritual. Those who shine too brightly cast shadows behind those around them, and from those shadows grow corruption and wickedness. Only two lights together, of different spirits, can truly illuminate the world.

2: A devil, playing with bags of ideas, once attempted to place a bag inside of itself, creating a paradox. The resolution of this paradox is responsible for the creation of the multiverse, and every world within it.

3: From foul chaos He was born, and through His power, the chaos was shaped, molded, and purified into the lands, seas, and skies we all know.

4:
```

Output:

![image](https://user-images.githubusercontent.com/585488/174668577-ec77fa6e-1f58-44b1-b6bb-f04db303908b.png)

