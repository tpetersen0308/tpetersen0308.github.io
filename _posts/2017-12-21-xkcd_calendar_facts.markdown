---
layout: post
title:      "XKCD Calendar Facts"
date:       2017-12-21 21:12:52 +0000
permalink:  xkcd_calendar_facts
---

![](https://imgs.xkcd.com/comics/calendar_facts.png)

One thing about learning to code is you start to see programs in everything. Like this [recent comic](https://xkcd.com/1930/) by XKCD. After reading it, I wanted to see if I could model the flow chart as a nested array structure containing strings that could then be used to randomly generate a given "fact" from the comic. Maybe I could even turn it into a fun little [CLI RubyGem](https://rubygems.org/gems/xkcd_calendar_facts). 

The really hard part was figuring out how to build a coherent string from the contents of an array that had no predictable structure. The array looks like this:

```
[
  'Did you know that ',
  [  
    ['the ', 
      ['Fall ', 'Spring '], 
    'Equinox '],
    ['the ', 
      ['Winter ', 'Summer '], ['Solstice ', 'Olympics ']
    ],
    ['the ', 
      ['earliest ', 'latest '], ['sunrise ', 'sunset ']
    ],
    ['Daylight ',
      ['Saving ', 'Savings '], 
    'Time '],
    ['Leap ', 
      ['Day ', 'Year ']
      ],
    'Easter ',
    ['the ', 
      ['harvest ', 'super ', 'blood '], 
    'moon '],
    'Toyota Truck Month ',
    'Shark Week '
  ],   

  [
    ['happens ', 
      ['earlier ', 'later ', 'at the wrong time '], 
    'every year '],
    ['drifts out of sync with the ', 
      ['sun ', 'moon ', 'Zodiac ', 'atomic clock in Colorado ', 
        [
          ['Gregorian ', 'Mayan ', 'Lunar ', 'iPhone '], 
        'calendar '
        ]
      ]
    ],
    ['might ', 
      ['not happen ', 'happen twice '], 
    'this year ']
  ],
    'because of ',
  [
    ['time zone legislation in ', 
      ['Indiana', 'Arizona', 'Russia']
    ],
    'a decree by the Pope in the 1500\'s',
    [
      ['precession ', 'libration ', 'nutation ', 'libation ', 'eccentricity ', 'obliquity '], 
      'of the ', 
      ['Moon', 'Sun', 'Earth\'s axis', 'Equator', 'Prime Meridian', 
        [
          ['International Date ', 'Mason-Dixon '], 
        'Line']
      ]
    ],
    'magnetic field reversal',
    ['an arbitrary decision by ', 
      ['Benjamin Franklin', 'Isaac Newton', 'FDR']
    ],
  ],
    "?\nApparently, ",
  [
    'it causes a predictable increase in car accidents.',
    'that\'s why we have leap seconds.',
    'scientists are really worried.',
    ['it was even more extreme during the ', 
      ['Bronze Age.', 'Ice Age.', 'Cretaceous.', '1990\'s']
    ],
    ['there\'s a proposal to fix it, but it ', 
      ['will never happen.', 'actually makes things worse.', 'is stalled in Congress.', 
			'might be unconstitutional.']
    ],
    'it is getting worse and no one knows why.'
  ],
    "\nWhile it may seem like trivia, it ",
    ['causes huge headaches for software developers.', 'is taken advantage of by high-speed traders.', 
		'triggered the 2003 Northeast Blackout.',
     'has to be corrected for by GPS satellites.', 'is now recognized as a major cause of World War I.']
]
```

Some "facts" only penetrate a couple of layers; some can only be built by following many forks in the path. I started simple--by making a less sophisticated structure and finding a way to build a coherent sentence out of that. Once I accomplished that, by writing code that would build a sentence out of an array with a predictable depth for each path, the next step was to figure out how to navigate a path through the array without having any idea how deep I'd be digging. 

![img](https://i.imgur.com/Ko0qUEm.jpg?3)

A little bit of whiteboarding helped. Once I figured out how to approach it, the logic wasn't too difficult. I realized that my algorithm would need to iterate through the array, and know how to handle an element based on its class and/or contents. Any given element could be one of four things: 
1. A string
2. An array of strings
3. An array containing string(s) and array(s) (any of which could again be any of these items but a string)
4. An array containging only arrays

If it was a string, it would be concatenated onto the existing string. If it was an array, I'd need to determine what it contained and handle it accordingly. I realized that I could determine whether it was (3) or (4) by calling #flatten on it and comparing its flattened length to its original length. If it was longer upon being flattened, it couldn't contain only strings. I would then need to sample from it and check the data type of that sample. If it was a string, it could be concatenated onto the "fact" and the iterator could move on to the next element of the master array. If it was an array, it could be any of the possible kinds of arrays and would be subject to the same logic that handles the master array--a case that could be handled relatively easily with recursion. At this point, all that was left to account for was the case of a simple array of strings, from which a sample could simply be concatenated. Implementing all of this logic in a FactGenerator class gave me the option of using instance variables. This simplified the recursion, because otherwise I would have had to make sure to pass the incomplete "fact" string back through each loop and ensure that it was not overwritten at any point. Gotta love OO programming!

```
class FactGenerator
  attr_accessor :fact
  
  def initialize
    @fact = String.new
  end
  
  def generate_fact(arr)
    arr.each do |el|
      if el.class == String
        @fact += el
      elsif el.flatten.length != el.length #if el has nested arrays
        unknown = el.sample 
        generate_fact(unknown) if unknown.class == Array
        @fact += unknown if unknown.class == String 
      else #array must be a one-D array of strings
        @fact += el.sample
      end
    end
  end
	
end
```

I built it all into a CLI program and pushed it up to [RubyGems](https://rubygems.org/gems/xkcd_calendar_facts) (the source code is on a [Github repo](https://github.com/tpetersen0308/xkcd_calendar_facts) as well), so you can install it with `gem install xkcd_calendar_facts`, run it with `xkcd_calendar_facts`, and read absurd, nonsensical, and IMHO hilarious "calendar facts" to your heart's delight. 

This was a fun independent project. It felt pretty satisfying to look at a problem, find a solution, and implement it successfully in code--particularly so because I really wasn't sure I had the chops to develop what initially felt like a pretty tricky chunk of logic. It also yielded an algorithm that could be applied to other similar situations which involve finding coherent routes through randomly structured arrays, which I think is pretty cool, though I still would like to see if I can re-factor it to accomplish the same task without recursion, or even yield every possible route. 

