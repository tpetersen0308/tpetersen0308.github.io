---
layout: post
title:      "React/Redux Portfolio Project: Birdable"
date:       2018-08-15 05:18:11 +0000
permalink:  react_redux_portfolio_project_birdable
---


I got into bird watching a few years ago while visiting a friend in Northern California. My friends (and girlfriend) constantly make fun of me when I have my binoculars hanging around my neck on hikes, or get distracted at the park, but there's something really special about identifying a species you've never seen before, and re-encountering your favorites can be like a visit with an old friend. One thing I'm still pretty bad at is recognizing birds by their song, so I was excited that the comparatively loose requirements for this project would allow me to tackle something I've been thinking about making for a while: a web app where people can practice identifying birds by their song. 

I was hoping to find an existing API that had images, taxonomic information, regional information, and song samples for common North American birds all in one place, but I couldn't find anything that really worked. The Audubon Society's online North American Bird Guide does have all of that information in one place, but that meant I was going to have to do a lot of scraping to make my own API using Ruby on Rails. The two most inconvenient parts of the Guide's setup were that regional and taxonomic data weren't available on the same filters, which meant it wasn't going to be doable in one scrape, and they display in an infinite scroll, which I had no idea how to handle before I had to do it. 

In order to build bird objects that had both regional and taxonomic information, I first scraped each taxonomic family in the filter menu and associated it with the `family_tid` property that is used as a query parameter in the url for the page that displays a given family's birds. I could then iterate through this hash to scrape each page by taxonomy, associating the individual birds with their families as I went. 

```
  def self.scrape_birds_by_family_tid(family_tid, family_name)
    url = "https://www.audubon.org/bird-guide?field_bird_family_tid=" + family_tid
    doc = self.get_page(url)
    birds = []
    doc.css("div.bird-card-grid-container div.page-0").map do |bird|
      next unless bird.css("div.field-name-field-bird-audio li a")[0]
      new_bird = {
        common_name: bird.css("h4.common-name a").text.downcase,
        scientific_name: bird.css("p.scientific-name").text.strip.downcase,
        image: bird.css("img")[0].attributes.values[0].value,
        song: bird.css("div.field-name-field-bird-audio li a")[0].attributes["href"].value,
        family: family_name.downcase
      }
      birds.push(new_bird)
    end
    return birds
  end
```

Getting the regional information was a little trickier, since enough birds are often associated with a given region that the infinite scroll would kick in before you could view them all on one page. 
I scraped the region names and `region_tid`'s similarly to how I scraped the families'. Then, By inspecting the requests that were fired when the scroll hit the bottom of the page, I found that it was making a request to a regular html page with an additional query parameter of `page_number`. I added this to the url that I was hitting with Nokogiri in a `while` loop that incremented the page number and scraped each page until it reached a page without any birds on it, before moving on to the next region's page. This method returned a hash of region names pointing at arrays of their respective birds' common names which I would be able to iterate through to associate each bird with its regions. 

```
  def self.scrape_bird_names_by_region(regions)
    birds_by_region = {}
    
    regions.each do |region_tid, region_name|
      
      page = 0
      url = "https://www.audubon.org/bird-guide?page=" + page.to_s + "&field_bird_family_tid=All&field_bird_region_tid=" + region_tid
      doc = self.get_page(url)

      while(doc.css("div.bird-card-grid-container div.page-" + page.to_s)[0])
        bird_names = doc.css("div.bird-card-grid-container div.page-" + page.to_s).map do |bird|
          next unless bird.css("div.field-name-field-bird-audio li a")[0]
          bird.css("h4.common-name a").text.downcase
        end
        birds_by_region[region_name] ||= []
        birds_by_region[region_name].concat(bird_names)
        page += 1
        url = "https://www.audubon.org/bird-guide?page=" + page.to_s + "&field_bird_family_tid=All&field_bird_region_tid=" + region_tid
        doc = self.get_page(url)
      end
    end
    return birds_by_region
  end
```

I rigged this all up in my seed file to persist all of the bird information, and created the necessary models, controllers and serializers to set up the API endpoint. This part of the process wasn't particularly easy or prohibitively difficult at any point--after a few failed attempts at seeding the database properly, I had the API set up exactly as I wanted and was ready to get started on my React/Redux frontend.

It's actually pretty surpring at this point how little time I spent on the functionality of the app's main purpose in comparison to things like getting all of my dependencies to load properly, setting up a proxy server to allow me to run the servers for my API and React app at the same time, or figuring out how to set up my `fetch()` calls in my actions such that I wouldn't get `TypeError`'s every time a component mounted. Honestly, getting the exercises to function the way I wanted was one of the easiest parts of this stage of building the app. 

What I knew I wanted was a single-page app with a home page that would display a randomly selected bird (really just to have any content on the home page at all), an practice page where a user could choose the set of birds they wanted to practice ID'ing from some dropdown menus of regions and families before getting started, and a browse page where one could filter birds in a similar manner to view more casually. The exercises would consist of images and names of four birds, randomly selected from the filtered list, and a song sample that matched one of the birds which the user could listen to and then try to match it to one of the birds. The solution would display instantly, including feedback on whether the choice was correct or not, and display the songs of all of the birds from the exercise. Users could continue practicing for as long as they like or call it a day whenever. 

Getting the dropdown filter menus set up involved a lot of mucking around in `react-bootstrap`. I knew I wanted to use it for styling, but there wasn't a built in component for a dropdown menu that displayed checkboxes from which a user could select (or deselect) as many options as they liked. This became one of those things that I labored over for hours, banging my head against my desk in frustration over why I couldn't get something that seemed so simple to work, only to sit down the next day, drop a `Checkbox` component into the `DropdownButton` component for each option, and have it work perfectly, even though I will go to my grave swearing this was the first, third, seventh, fifteenth, and ninety-third thing I had tried the day before to absolutely no avail. I allowed myself about fifteen minutes to pout about it and moved on. 

I rigged up an event handler to each check box that would update my `BirdsFilter` state with the user's (de)selections:

```
  handleFamilyCheckbox = event => {
    if (event.target.checked) {
      this.selectFamily(event.target.value);
    } else {
      this.deselectFamily(event.target.value);
    }
	}
	
  selectFamily = family => {
    this.setState({
      ...this.state,
      selectedFamilies: this.state.selectedFamilies.concat(family)
    }, () => console.log(this.state.selectedFamilies))
  }
```

which I did quite similarly for the region menu, and set up some methods to select birds from the store based on the user's filter choices:

```
  handleSubmit = event => {
    event.preventDefault();

    let birds = this.props.birds;
    birds = this.filterByFamilies(birds, this.state.selectedFamilies);
    birds = this.filterByRegions(birds, this.state.selectedRegions);

    this.props.selectAction(birds);

    this.props.handleSubmitRoute();
  }
	
  filterByFamilies = (birds, families) => {
    if (families.length > 0) {
      birds = birds.filter(bird => families.includes(bird.family))
    }
    return birds
  }

  filterByRegions = (birds, regions) => {
    if (regions.length > 0) {
      for (let regionName of regions) {
        birds = birds.filter(bird => bird.regions.map(region => region.name).includes(regionName))
      }
    }
    return birds
  }
```

One thing that was getting really annoying at this point was that every time I saved and tested my changes, I had to make sure I was waiting for the async action to get the birds from the API was complete before hitting the submit button, or else I would get a `TypeError` when my `Exercise` component subsequently mounted and started calling functions on yet-to-be-present objects. I handled this by passing a `loading` prop in from the store with a boolean value that I could use to conditionally display a loading message until the async was complete. It took me a while to get this working too, but I eventually solved it by setting `loading`'s initial state to `true`, and updating it to `false` when the async finished. For some reason, dispatching a `"LOADING"` action in my `fetch()` call before the request to the API wasn't working for this as I'd understood it should from the documentation, but I spent a lot of time trying to debug that issue before taking the easy way out. I've just assumed, for the time being, that the components that threw errors were getting mounted before the `"LOADING"` action was dispatched. 

Getting the exercises to flow properly went fairly smoothly. An exercise had the following structure in state:

```
exercise: {
  birdSelection: [],
	problem: {...},
	userAnswer: null,
}
```

The `Exercise` component would render a `Problem` or `Solution` component conditionally, based on whether or not the `userAnswer` was set to `null`. When a new exercise was displayed, it would be `null`, because a user hadn't submitted an answer yet. If it was set to a number (which would be associated with the bird object the user judged to be the correct answer), a `Solution` component would be rendered. Upon clicking "Next Exercise," it would be updated to `null` in the store so the an updated `Problem` component would be rendered, and so on. 

This is something that I would really like to make available on Heroku, but I have some more features I'd like to add--which fell too far outside the scope of the requirements for this project for me to focus on yet--before I do. I want to add additional exercise types for matching names to bird images or songs, as well as an exercise where one can view a single bird and select from a list of songs to identify it by. I'll also add user profiles, so people may sign in with Google or Facebook and track their progress. Finally, I'm going to start recording how frequently each bird species is associated with correct and incorrect answers, so I can calculate a solve rate that should enable me to include difficulty settings as well as filtering by commonness of species, because I expect those two to be in direct correllation. As always, the repo can be viewed on [GitHub](http://github.com/tpetersen0308/birdable) by anyone who's curious, and I've also posted a brief [video demo](https://youtu.be/WWgPOrNw2ZQ) on YouTube. Happy birding!
