---
layout: post
title:      "Bad Habits And The Sieve Of Eratosthenes"
date:       2017-11-08 20:24:38 -0500
permalink:  bad_habits_and_the_sieve_of_eratosthenes
---


Something I've been repeatedly getting myself into trouble with in my still-nascent experience in programming is trying to polish as I go. I haven't been in the online-web-developer program at Flatiron for very long, but I've already seen the same kind of advice repeated in the Ask A Question section and Flatiron-hosted study groups: get the code working, then refactor. It seems to be a foundational tenet of TDD, and the inherent wisdom of it isn't mysterious. Get it working before you worry about how pretty it is. Function before form. I'm still terrible at it. 

So far, I haven't been struggling too hard with the curriculum. My CLI data gem project is fast approaching though, so I may be singing a different tune before long. For most of the labs, I don't have to refactor much once I get the code working, so I don't really notice myself getting more deeply entrenched in my bad habit. Then I hit something a little more challenging, and struggle for way longer than I need to before I realize how much harder I'm making it on myself by focusing on writing pretty code while I'm still trying to wrap my head around how to just get the tests to pass. 

Yesterday, someone on Slack brought up the [Sieve of Eratosthenes](http://en.wikipedia.org/wiki/Sieve_of_Eratosthenes)--a method for finding all of the prime numbers less than or equal to a given number--and suggested that it would be a good challenge to implement it in code. I was busy doing a clearance pass of the latest episode of the reality TV show I work for, so it was perfect timing. It took me about an hour, about halfway through which I realized I really just needed to see if I could even get something (anything) working that modeled this process, so I stopped trying to be fancy, and wound up with this:

```
def  sieve_of_eratosthenes(n)
  doors = Array.new(n, "open")
  doors[0] = "closed"
  counter = 2

  until counter == n
    close_doors_by_multiple(doors, counter, n)
    counter += 1
  end

  primes = doors.collect.with_index { |door, index| door == "open" ? index + 1 : next }

  primes.delete_if { |element| element == nil }

end

def close_doors_by_multiple(doors_array, multiple, ceiling)
  multiples = Array.new(ceiling/multiple.floor)
  multiples = multiples.collect.with_index do |element, index|
    element = multiple*(index + 1)
  end
  multiples.shift
  multiples.each do |element|
    next if doors_array[element - 1] == "closed"
    doors_array[element - 1] = "closed"
  end
  doors_array
end
```

Keep in mind, I wasn't simply triyng to populate an array with the primes up to n, but actually mimic the logic of the Sieve. It works, but it's a lot of code. I couldn't get it right by trying to be really mathy and abstract from the start, so instead I just tried to model the visualization that was suggested as a means of understanding the Sieve: Imagine 100 lockers, all with their doors open. Close the first one. The next open locker is #2, so close every other door after that. The next open one is #3, so close every third one after that. The next open locker is #5, so close every fifth one after that, and so on until you can't close anymore doors. All of the doors left open represent all prime numbers less than or equal to 100. 

I had to do my job for a while after that, and then study some of the actual Flatiron curriculum, but knowing I could acheive the same result with more elegant code was bugging me, so I returned to it today and came up with this:

```
def seive_of_eratosthenes(n)
  range = (2..n).to_a
  primes = []

  until range.size == 0
    primes << range.shift
    range.delete_if do |number|
      number % primes.last == 0
    end
  end

  primes

end
```

I made some minor tweaks after that. I realized that starting the range at 3 makes more sense, because 2 will always be included in the collection of primes, but then I had to shovel the first element of the range array onto the primes array *after* iterating through the range or else the method would include 4 with the primes. I put the iterator on one line and this is what I landed on:

```
def sieve_of_eratosthenes(n)
  range = (3..n).to_a
  primes = [2]

  until range.size == 0
    range.delete_if { |number| number % primes.last == 0 }
    primes << range.shift
  end

  primes
end
```

It's almost the same process as the first version, just way more abstract and way more efficient. I spent a little more time trying to come up with a way to use a higher-level iterator in place of the #until block, but ultimately decided there's not really anything appropriate for an iterator to iterate through for that purpose. 

I want to give myself more credit than to say I'd never have come up with this had I not trudged through it in its most explicit form the first time, but I can definitely say it would've taken me way longer. And, because I took the less abstract route the first time, I feel like I came out with a better understanding of the intuitive logic behind the method--that if a given number is prime, no multiple of it can also be prime--which then made it easier to apply in more abstract code. Now, looking back, It's clear that if I'd simply started with the goal in mind of just getting the code working, and then prettied it up once I worked out all the kinks and truly wrapped my head around it, I would've saved a lot more time and brain-power. 
