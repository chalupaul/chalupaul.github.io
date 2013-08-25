---
layout: post
title: "ruby games"
date: 2013-08-25 12:23:03
categories: 
---

Did you ever use RPG maker as a kid? It was a program where you could make tiny little sprite based RPGs. It had about every feature you could want. Tiled sprites,  map layers, characters, spells and items, dialog systems, and most importantly--a way to customize all sorts of things in "rpg script."

I think it's actually how I learned to program, though I barely understood that at the time. I think I was 11 or 12, you know, right at that age where you decide really important things that shape your future like "Am I interested in computers, or football?" 

One of the neatest things about RPG maker was that it was a bootleg for us English speakers. A guy who went by Don Miguel (he is incidentally russian) translated the runtime to English. And it created a huge community of modelers, game makers, musicians, rippers (people who extract from super nintendo games and distribute them). 

RPG Script was really well executed. Definitely the best feature in the program. It was a perfect example of making complex progarmming possible to newcomers. You would interface with it by clicking buttons which would open dialog boxes. And it would output RPG script in a little console so you could learn how it works. And you could make all sorts of intricate game systems that nobody would ever see in commercial games (probably becaues they were way too non-user friendly). 

Games that people released were by and large tech demos. They would feature elaborate custom combat systems, incredible spell casting systems, etc. I wrote a banking system complete with retirement savings accounts and mortgages for example, and I had a true to life food/comfort system that I subjected players to: Run around too much without drinking or eating? You're going to start losing stats. Take too long and you would crawl rather than walk. Too tired, and dialogs with people would be incomphrehensible. My oh my, nobody would play that game.

It was amazing to see the community offering what I now know to be such good programming advice, mainly because a system where you click to code punishes you harshly if you don't make things easy on yourself.

Here's an example. I was implementing a crafting system where you could trap creatures, fish, hunt, and forrage for materials. Then taking them back to the village, certain vendors would would only buy certain items. I always thought it was weird that a weapons vendor would be interesting in buying beaver tails, and why a healer would be interested in iron bars. 

So the way you would solve this problem in RPG maker was to switch items in your players inventory for mirror items that have 0 cost (0 cost items cannot be sold). That way you would only replace items the vendor wouldn't buy. When the shopping dialog was done, you do the switch again and the player is none the wiser.

Obviously there are some interesting code design challenges in there which may make another post later. But suffice to say, if you just hardcore replaced every item with a duplicate, you would never be able to add other items to the game without making changes here too which is a nightmare. You absolutely would have to determine the item you should replace bidirectionally to make this livable.

So a very excellent friend of mine grabbed a new copy of [rpg maker vx ace][rpgmaker] and started making this really cool game playfully mocking japanese school-girl stuff. It's pretty hilarious really in a tongue and cheek sort of way. But he had the idea of mana points (MP) being stored directly on the items instead of a generic pool that gets derived by other stats like Intellect or Cuteness.

So there are a few interesting things you can do with his system. First, you need to decay MP from different items it a way that makes sense. That means that every item needs to degrade at the same rate, with edge cases taking extra MP from the "fullest" items. Also, you should never take the last MP from an item unless there's no choice. In this way, you could have a system where items grant your character traits, or even spectacular spells that are powered by their own MP batteries. But casting any spells gradually reduces that pool. Then you have to make the choice between keeping the traits, using the item spell, or casting generic spells, etc. You could even have certain enemy AI that would mana burn those particular items to disable their traits. That would make for a great combat system I think.

Mike (my friend) and I talked about it a bit over facebook, and I decided to write a prototype of his model. The newest version of RPG Maker lets you write directly in Ruby, which basically means you can make any system you have ever wanted to see. It's an innovator's and tinkerer's paradise--a perfect game-making sandbox that doesn't mean you have to have an army of artists creating assets round the clock.

This prototype isn't a drop-in module for RPG maker, it's just a little script showing what you could do with it. Comments in the code show why it does what it does and the design decisions taken into account.

{% highlight ruby linenos=table %}
#!/usr/bin/env ruby


class Item
  attr_accessor :name, :mp, :max_mp
  def initialize(name, mp, max_mp)
    @name = name
    @mp = mp
    @max_mp = max_mp
  end
end

=begin 
This function takes a number and creates a recursive array of halfs of it.
So it takes half and adds it to the array (rounding up), then it takes the 
rest and halfs it again with the same logic. So halfer(15) would give you:

[8, 4, 2, 1]
=end 
def halfer(n)
  if n == 1
    return [n]
  else
    h1 = n / 2
    h2 = (h1 * 2 == n)? h1 : h1 + 1
    return [h2] + halfer(h1)
  end
end

=begin
This function applies an array of mp costs to an array of items.
It assumes that the list of items are pre-sorted. Since it's 
somewhat recursive, you don't want to sort at every run unless 
must. It's possible that you would try and reduce an item by
more MP than it has, so we just halfer() it again, and resort
the costs array and try again.

This is why above, you do not want to take more MP cost than 
available MP to use--it would hit max recursion.
=end

def shrink_mp(items, costs)
  if costs.empty?
    return items
  else
    if items[0].mp >= costs[0]
      item = items.shift
      cost = costs.shift
      item.mp -= cost
      items << item
    else
      cost = costs.shift
      costs = costs.concat(halfer(cost)).sort.reverse
    end
    shrink_mp(items,costs)
  end
end

get_total_mp = lambda {|is| is.inject(0){|sum,n| sum + n.mp}}

# Equip our character with some simple gear.
items = {
    shirt: [1,10], 
    pants: [1,5], 
    belt: [0,0], 
    hat: [1,1], 
    weapon: [1,20]
}.map {|k,v| Item.new(name=k.to_s, mp=v[0], max_mp=v[1])}

# Pretend to cast a spell (costs 3MP). Don't change this
# to a number bigger than your mp pool. ;)
cost = 3

=begin
Get pre and post results size so we can deal with remainders. Since 
percentage values can end up in decimals, for very large or small 
numbers you end up with rounding errors that leave MP cost on the 
table and need to be dealt with. So a simple check to make sure we 
took as much mana as the spell costs and deal with the results.
=end
current_total_mp = get_total_mp.call(items)

items.map do |x| 
  val = ((x.mp.to_f / items.inject(0){|sum,n| sum + n.mp}) * cost).to_i
  x.mp = x.mp - val >= 0? x.mp - val : 0
end

new_total_mp = get_total_mp.call(items)

remaining_mp_cost = (current_total_mp - new_total_mp - cost).abs

if remaining_mp_cost > 0
  # Only items with mana value are needed, and you can't divide by 0 
  # and expect much success
  items_filtered = items.select{|item| item.max_mp > 0}
  # Order by highest percentage full and highest mp. In this way, you never 
  # cheat one item when there's a more appropriate one.
  items_filtered = items_filtered.sort_by { |item| 
    [( item.mp / (item.max_mp * 0.01) ), 
    item.mp]}
  # Finally apply the remaining MP costs
  _ = shrink_mp(items_filtered,halfer(remaining_mp_cost))
end
{% endhighlight %}


[rpgmaker]: http://www.rpgmakerweb.com/products/rpg-maker-vx-ace