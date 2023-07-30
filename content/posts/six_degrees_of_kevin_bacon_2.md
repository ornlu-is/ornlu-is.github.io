---
title: "Six Degrees of Kevin Bacon 2: Web Scraping Movie Data"
date: 2023-07-14T12:17:55+01:00
categories: ["Six Degrees of Kevin Bacon"]
description: "Web scraping movie and actor information to build a graph of all actors, connecting them via movies they worked together on."
draft: true
---

To achieve the goal of building a social graph for movie actors, the first step is to gather data. For that matter, I will be scraping data from the [Open Media Database (OMDB)](https://www.omdb.org), which is a free non-commercial database for film media.

{{< admonition type=tip title="Other posts in this series" open=true >}}
* [Six Degrees of Kevin Bacon 1: Introduction](https://ornlu-is.github.io/six_degrees_of_kevin_bacon_1/)
* [Six Degrees of Kevin Bacon 3: Collecting the Data](https://ornlu-is.github.io/six_degrees_of_kevin_bacon_3/)
{{< /admonition >}}

## A plan for crawling OMDB

The OMDB is organized very conveniently for my purposes. The website has an encyclopedia section where we can pick any given year and it will list all movies in the chosen year,*e.g.*, the page for the year 2023:
* https://www.omdb.org/en/de/encyclopedia/year/2023/index/1 

The link above takes us to the first page containing media from 2023. This link also gives us a hint on how we might crawl the website. More specifically, the `/year/2023/` part of the link looks *very* convenient. If we change this to, say `/year/2022/`, we get the first page of movies for 2022! In short, we have a very easy of navigating years, we can just manually pick them (and we should be safe because it takes a whole year for a year to elapse). Additionally, the link terminates with `/index/1`. At the bottom of this page, we have the option of navigating to the next set of movies that came out in 2023:

{{< figure src="/images/six_degrees_of_kevin_bacon_2/omdb_next_page.png" title="Pagination in OMDB" >}}

Clicking the box for the second page, takes us to a new webpage with the link:
* https://www.omdb.org/en/de/encyclopedia/year/2023/index/2

Convenience galore, all that changed in the link was just the last number!

Naively, we might think that a good plan for iterating over the pages for a given year would be to start at the first page, collect the information required, find the link for the next page, navigate to it and then repeat the process until we run out of pages. To understand why this probably isn't the most viable option, let us inspect the pagination element:

```html
<div id="pagination" class="pagination">
   <ul>
      <li class="disablepage">
         <span>
         «&nbsp;previous
         </span>
      </li>
      <li class="currentpage">
         <span>
         1
         </span>
      </li>
      <li>
         <a href="/en/de/encyclopedia/year/2023/index/2">
         2
         </a>
      </li>
      <li>
         <a href="/en/de/encyclopedia/year/2023/index/3">
         3
         </a>
      </li>
      <li>
         ...&nbsp;
         <a href="/en/de/encyclopedia/year/2023/index/18">
         18
         </a>
      </li>
      <li class="nextpage">
         <a href="javascript:void(0);" id="nextpage">
         Next&nbsp;»
         </a>
      </li>
   </ul>
</div>
```

As we can see, the `<li class="nextpage">` contains an anchor element but this element's `href` attribute does not contain a link. But convenience is on our side again because the pagination in OMDB always shows the last possible page we can navigate to! This means that, to get all pages for a given year, all we have to do is scrape the first one and extract the number of the last page. Then, the links all pages in between the first and last page can be obtained by simply incrementing `/index/1` until we reach `/index/<lastpage>`.

Note that the list item for the last page does not contain any attribute that allows us to distinguish it from all others, but there is one list item that does, and that is the one corresponding to the next page, which always shows up after the link for the last page and has the attribute `class` with the value `nextpage`. So, in essence, we just need to locate the `div` element that has an `id` attribute with value `pagination` and, in its child elements, locate the `li` element with the attribute `class` and value `nextpage` and then get its previous sibling, that holds the information on the last page.

## Getting the movie links

We have a plan to get all the pages for all the years, but we have yet to obtain the actual movie links. Fortunately, in each page, there is a `div` element with the attribute `id` and value `results` which holds, you guessed it, the movies in the page! For the last page of 2019, that HTML element looks like this:

```html
<div id="results">
   <div id="search-listing-70" style="float: left; overflow:hidden;">
      <div id="items-for-page-70">
         <div id="movie_search_result_172562" class="result-box">
            <div class="image">
               <a href="/en/de/movie/172562-kase-und-blei">
               <img alt="" src="https://www.omdb.org/images/misc/no-thumbnail.png?1319773240" width="92">
               </a>
               <br>
               <div class="votebar">
                  <div class="mediumvote" title="0.00">
                     <div style="width: 0.00%">
                     </div>
                  </div>
               </div>
            </div>
            <div class="link">
               <a href="/en/de/movie/172562-kase-und-blei" title="Movie: Käse und Blei">
               Käse und Blei
               </a>
               <div class="small">
                  2019
               </div>
            </div>
         </div>
      </div>
   </div>
</div>
```

The HTML element we can easily scope out is

```html
<div class="link">
   <a href="/en/de/movie/172562-kase-und-blei" title="Movie: Käse und Blei">
   Käse und Blei
   </a>
   <div class="small">
      2019
   </div>
</div>
```

because we can identify it via the `class` attribute that has the value `link`. Then, all we need to do is fetch the value of the `href` attribute of the anchor element to get the portion of the link that we have to append to `https://www.omdb.org` to get a movie link and, additionally, we can also extract this element's data to get the movie's name.

There is one last detail about extracting a movie link: sometimes, OMDB another hyperlink associated with each movie. If you look closely at the screenshot above, you can see that some movies' name is prefixed by a small arrow. Unfortunately, this small arrow has the exact same hyperref value as the link we are interested, meaning that we get duplicated links. While we could solve this with a slightly more complicated scraping strategy, we can also tackle this by implementing deduplication which will be sufficiently efficient since the number of movies per page is small (max. 24).

## Extracting cast data

Now that we have a way to get all the links for all movies, we have to think about how we are extracting the cast. When we navigate to a given movie's link, we see that there is a clickable tab named "Crew/Cast". Clicking on it takes us to a page whose link is the same as the movie's link but affixed with `/cast`, *e.g.*

* https://www.omdb.org/en/de/movie/67669-how-to-train-your-dragon-the-hidden-world/cast

and is filled with information on who worked on the movie. Particularly, there is a section named "Actors" where the data we are looking for is. Inspecting this element we get:

```html
<div id="movie-list-4" class="list" style="margin:0px; clear:left;">
    <div class="list-view-headline">
       <a class="edit-button" href="#" id="link-sort-movie-4" onclick="new Ajax.Request('/en/de/movie/67669/activate_cast_sorting?type=movie-4', {asynchronous:true, evalScripts:true}); return false;" title="Sort this list">sort</a>
       <a class="edit-button" href="#" onclick="try { box = new lightbox('/en/de/cast/edit?department=4&amp;movie=67669-how-to-train-your-dragon-the-hidden-world', false, true) } catch (e) {}; return false;" title="edit">edit</a>
       <a name="department_4"></a>
       <h3>Actors</h3>
    </div>
    <ul id="movie-4" class="sortable-list">
       <li id="actor_885986">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/449-jay-baruchel" title="Person: Jay Baruchel">Jay Baruchel</a></strong>
          </div>
          as
          <span class="bold">
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/885986/edit_character', false, true) } catch (e) {}; return false;">Hiccup</a>
          (Voice)
          </span>
       </li>
       <li id="actor_885987">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/59174-america-ferrera" title="Person: America Ferrera">America Ferrera</a></strong>
          </div>
          as
          <span class="bold">
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/885987/edit_character', false, true) } catch (e) {}; return false;">Astrid</a>
          (Voice)
          </span>
       </li>
       <li id="actor_967703">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/1164-f-murray-abraham" title="Person: F. Murray Abraham">F. Murray Abraham</a></strong>
          </div>
          as
          <span class="bold">
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967703/edit_character', false, true) } catch (e) {}; return false;">Grimmel</a>
          (Voice)
          </span>
       </li>
       <li id="actor_885988">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/112-cate-blanchett" title="Person: Cate Blanchett">Cate Blanchett</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/885988/edit_character', false, true) } catch (e) {}; return false;">Valka</a>
          (Voice)
       </li>
       <li id="actor_885992">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/17276-gerard-butler" title="Person: Gerard Butler">Gerard Butler</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/885992/edit_character', false, true) } catch (e) {}; return false;">Stoick</a>
          (Voice)
       </li>
       <li id="actor_885989">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/24264-craig-ferguson" title="Person: Craig Ferguson">Craig Ferguson</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/885989/edit_character', false, true) } catch (e) {}; return false;">Gobber</a>
          (Voice)
       </li>
       <li id="actor_885990">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/21007-jonah-hill" title="Person: Jonah Hill">Jonah Hill</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/885990/edit_character', false, true) } catch (e) {}; return false;">Snotlout Jorgenson</a>
          (Voice)
       </li>
       <li id="actor_967704">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/54691-christopher-mintz-plasse" title="Person: Christopher Mintz-Plasse">Christopher Mintz-Plasse</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967704/edit_character', false, true) } catch (e) {}; return false;">Fishlegs</a>
          (Voice)
       </li>
       <li id="actor_967705">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/41091-kristen-wiig" title="Person: Kristen Wiig">Kristen Wiig</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967705/edit_character', false, true) } catch (e) {}; return false;">Ruffnut</a>
          (Voice)
       </li>
       <li id="actor_885991">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/157561-kit-harington" title="Person: Kit Harington">Kit Harington</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/885991/edit_character', false, true) } catch (e) {}; return false;">Eret</a>
          (Voice)
       </li>
       <li id="actor_967706">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/224617-robin-atkin-downes" title="Person: Robin Atkin Downes">Robin Atkin Downes</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967706/edit_character', false, true) } catch (e) {}; return false;">Ack</a>
          (Voice)
       </li>
       <li id="actor_967707">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/182547-kieron-elliott" title="Person: Kieron Elliott">Kieron Elliott</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967707/edit_character', false, true) } catch (e) {}; return false;">Hoark</a>
          (Voice)
       </li>
       <li id="actor_967708">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/224928-julia-emelin" title="Person: Julia Emelin">Julia Emelin</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967708/edit_character', false, true) } catch (e) {}; return false;">Griselda the Grevious</a>
          (Voice)
       </li>
       <li id="actor_967709">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/35981-gideon-emery" title="Person: Gideon Emery">Gideon Emery</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967709/edit_character', false, true) } catch (e) {}; return false;">Trapper</a>
          (Voice)
       </li>
       <li id="actor_967710">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/55398-ashley-jensen" title="Person: Ashley Jensen">Ashley Jensen</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967710/edit_character', false, true) } catch (e) {}; return false;">Phlegma</a>
          (Voice)
       </li>
       <li id="actor_967711">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/173649-olafur-darri-olafsson" title="Person: Ólafur Darri Ólafsson">Ólafur Darri Ólafsson</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967711/edit_character', false, true) } catch (e) {}; return false;">Ragnar the Rock</a>
          (Voice)
       </li>
       <li id="actor_967713">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/83216-james-sie" title="Person: James Sie">James Sie</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967713/edit_character', false, true) } catch (e) {}; return false;">Chaghatai Khan</a>
          (Voice)
       </li>
       <li id="actor_967714">
          <div class="freeze handle"><img alt="Sort" class="sort" src="https://www.omdb.org/images/icons/sort.png?1319773240" width="25" height="11"></div>
          <div class="person">
             <strong><a href="/en/de/person/20049-david-tennant" title="Person: David Tennant">David Tennant</a></strong>
          </div>
          as
          <a class="edit" href="#" onclick="try { box = new lightbox('/en/de/cast/967714/edit_character', false, true) } catch (e) {}; return false;">Spitelout / Ivar the Witless</a>
          (Voice)
       </li>
    </ul>
</div>
```

Examining the HTML above, we can clearly see that the cast of a movie is always stored under a `div` element with the `id` attribute with the value `movie-list-4`. That is great! Inside this element, we can see that each actor's name and link to their page is contained inside list item elements. However, the `id` attribute for these elements isn't unique for each actor, which means that we'd have to do some pattern matching if we were hellbent on using it to identify each actor. Another approach, which is the one I will take, is to notice that inside each of the aforementioned `div` elements, there is another `div` with the `class` attribute valued `person` inside which there is a single anchor element with not just the actor's name as data, but also the actor's OMDB webpage link, which we can then use to safeguard against actors with the same name.

## Next step

Now that we have a functional description on how to scrape the data we want, we need to come with an implementation plan for how we are going to pratically achieve this goal. Check out my plan for it in the next post of this series:

* [Six Degrees of Kevin Bacon 3: Collecting the Data](https://ornlu-is.github.io/six_degrees_of_kevin_bacon_3/)
