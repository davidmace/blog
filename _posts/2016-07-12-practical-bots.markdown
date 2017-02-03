---
layout: post
title: Practical Bots
date: 2016-07-12 15:46
comments: true
external-url:
categories: Everything
---

.

# 0. Goals

The goal of this minibook is to teach a software engineer how to build a production-grade semi-automated natual language bot. This framework produces highly accurate responses (~95% accuracy on reasonable data), is not language-specific (ie. English, French, Japanese, etc), improves automatic response rate over time with more data, is flexible to a large variety of domains, and creates highly interpretable models. I include code for all of the components you'll need to create a basic bot so you can pick and choose which ones you need for your specific task.

I assume basic knowledge of services and high-level machine learning concepts (ie. training set, feature space). You won't need to know any of the math behind the machine learning models. There is no math in this tutorial. Only code and concepts.



# I. Understanding the Problem and Data

## A. What is a Bot Engine?

A bot provides semi-automatic responses to a user's text requests. Except in very simple cases, natural language processing technology is not yet good enough to answer 100% of user requests. This is why bot engines, including our framework below, often leverage human responders for a portion of hard tasks.

Our bot engine works by 1. converting a user's input text into a structured format, 2. calling APIs to get and update background variables, and 3. responding with a filled-in template. First look at the examples below then we'll break this down in more depth in the next section.

### Example 1<br>
**User:** Find me a place to buy a Boston Terrier near Boston.<br>
**Background API Call:** locallookup( location=Boston, item=Boston_Terrier) -> (id=LOC/125667, name=Jim's_Dog_Shelter)<br>
**Bot:** How about this: Jim's Dog Shelter.<br>
**User:** Thanks. Can you check if they're open tomorrow?<br>
**Background Function Call:** is_open( id=ENT/125667, day=1/2/2017) -> True<br>
**Bot:** Yes!<br>

### Example 2<br>
**User:** How many days until my package arrives?<br>
**Background Function Call:** arrival_date(user_id=USER/13578) -> Error: multiple_packages<br>
**Bot:** Which order?<br>
**User:** my Kindle<br>
**Background Function Call:** arrival_date(user_id=USER/13578, specifier=Kindle) -> 1/4/17<br>
**Bot:** It's expected on January 4th!<br>


## B. Our Engine's Architecture

There are five major components of our engine.

### 1. Parameter Extraction
In the examples above, "Boston", "Boston Terrier", and "Kindle" are parameters. They are specific values that we need to pass to our API calls. Before the word correction phase (step 2), we extract regex entities (emails, urls, phone numbers, numbers, and datetimes) and proper nouns (names and locations). We still need to correct the spelling of custom entities before extracting them, so we wait until step 3 to do this.

### 2. Word Correction
Users often misspell words and use colloquial language (ie. ty, plz, k). It's important that we correct these errors before our algorithm attempts to understand the syntax and extract custom class parameters (ie. colors, burger toppings).

### 3. Custom Parameter Extraction
Often a bot needs to recognize custom classes: for example color parameters (ie. red, green, purple) or room prices. This section details a simple architecture for writing and managing these custom entity lists. I also walk through a clever technique to efficiently spellcheck massive (10k-1m length) lists of custom entities (ie. hotel names, book titles, rock bands).

### 4. Text Classification
Each of the user's inputs corresponds to a function call with parameters. In the examples above, the function calls include local_lookup, is_open, and arrival_date. Assuming we have N preset function calls, we need to classify the function call of each new user input. This is very difficult. Syntax and word choice vary tremendously between different user inputs corresponding to single function call (ie. hi, good morning, warm regards). We use human labellers to label the function call of any input for which our algorithm returns low certainty in its answer.

### 5. State Management
Requests often depend on previous information in the conversation. Parameters are passed between function calls. In this section, I detail a technique for managing this state information. This is definitely the most open-ended and application-specific section. Unlike with the components above, you may have to fiddle with these techniques to get them working on your use case.

## C. Data You Expect vs Data You Get

As a developer, you know it's hard to predict how users will interact with your UI. Natural language inputs have many more degrees of freedom than traditional UI inputs (ie. click, scroll, etc). It's extremely difficult to wrap your head around the types and complexity of your user's requests without actually digging into the data.

Your data will surely present its own challenges, but below are a few real-world examples of common complexities your bots will have to resolve.

### Example 1: Spelling and Grammar
**My Expected Input:** I would like to send apples Monday night after 3pm.<br>
**User Input:** hi sir please i like to deliver apple monday night aftr 3 PM<br>
**Analysis:** Our system has to perform under spelling and grammar errors. In one of my normal datasets from the United States, user inputs contain incorrect spelling ~30% of the time and incorrect grammar ~20% of the time. In one of my normal English datasets from Southeast Asia, these numbers jump to ~40% spelling errors and ~40% grammar errors.

### Example 2: Colloquialism
**My Expected Input:** May I please have the usual arrival time for tomorrow.<br>
**User Input:** may i plz know expected arrival tym for tom<br>
**Analysis:** Users often use shorthand phrases like "tom" for "tomorrow" or "ty" for "thank you". Also language is very imprecise. Without context, it's difficult to know if the user is talking about a person "Tom" or is using shorthand for "tomorrow".

### Example 3: Complexity
**My Expected Input:** I am having trouble submitting my order.<br>
**User Input:** I cant submit my order. what should I do? Shipping is selected but I get an error saying "please select a shipping option".<br>
**Analysis:** Users often give you far more detail than you'd like. In this case, it's difficult to decipher whether to fall back on a stock "email support" answer or more specific answers on "shipping options".

### Example 4: Constraints
**My Expected Input:** Please find me a local restaurant in Hawaii.<br>
**User Input:** Im looking for a local restaurant. We are in Hawaii. I'd prefer a place that serves salmon.<br>
**Analysis:** Users have all sorts of weird constraints. First of all, it's difficult to linguistically recognize all of the odd conditions: "that serves salmon", "that costs less than $$", "within a mile of my location", etc. Even more importantly, users are going to surprise you with new types of constraints that your system has never seen (ie. "find me a hotel that's within walking distance of the beach and a grocery store").

### Example 5: Complexity++
**My Expected Input:** How does my son sign up for your school's classes?<br>
**User Input:** My son is a student at Ohio State. He is a rising senior. I would like to enroll him in summer courses at your institution. Does he just sign up online, or should he go to an info session, or do I need to chat further with you?<br>
**Analysis:** You might be able to answer really long and complex inputs with a stock answer. However, you're probably better off guiding your users to enter concise and direct inputs rather than trying to improve your technical solution. 

### Example 6: Answers Requiring Reasoning
**Previous Bot Output:** Are you an employee of the company or a prospect?<br>
**My Expected Input:** employee<br>
**User Input:** I have worked for you guys for 20 years.<br>
**Analysis:** A human knows logically that if the user "works for you", then they're your employee. A computer does not. Algorithms are good at classifying phrase structures they've seen before, not reasoning out relationships.

### Example 7: Mixing Answers and Questions
**Previous Bot Output:** Which city would you like to book the hotel in?<br>
**My Expected Input:** Lisbon.<br>
**User Input:** Lisbon, can you make sure it's pet friendly<br>
**Analysis:** Users mix answers and new requests in a single response. A good system needs to first store "Lisbon" as the previous question's response then process the follow-up request "can you make sure it's pet friendly". Splitting inputs into multiple requests/answers is a very hard unsolved task.

### Example 8: Missing Context
**Previous Bot Output:** Your book order has shipped.<br>
**My Expected Input:** When is my book order arriving?<br>
**User Input:** When is it getting here?<br>
**Analysis:** New requests have to be taken in the context of past inputs. Our system needs to understand that "it" corresponds to the book order mentioned in the previous bot output.


## D. Lots and Lots of Esoteric Edge Cases

Natural language data is extremely varied in syntax and word choice. Below is an example to convince you that, regardless of how varied you think natural language is, you're probably underestimating its complexity.

The graph below is the distribution of phone number formats from one of my real-world datasets. Phone number formats (ie. (111) 111 1111 and 111-111-1111) are much more consistent than other natural language tasks like "ways of saying I need X". However even in this simple case, 16 different formats appear in my sample of 30 phone numbers. Handling only the top 5 formats would miss ~50% of cases (ie. +11 111 111-1111 ext. 11). 

{:refdef: style="text-align: center;"}
<img src="/assets/tele-distribution.png"/>
{: refdef}

Let this be a reminder of the complexity of your users' inputs.

# II. Extracting Parameters


## A. Emails

Email extraction is easy because the format <kbd>x@y.z</kbd> rarely if ever picks up false positives.

Here's the regex:
```
\b[\w\.\-!#$%&\'*\+\/=\?\^_`{\|}~]+@[\w\.\-]+\.[\w\-]+\b
```

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; The extract_emails function of <a href="https://github.com/davidmace/practicalbots">bot_helpers.py</a> contains code for this task.

And some simple test cases:

```
a924-g@gmail.com, true
david-c.mace@hot-mail.com, true
send @ 5pm.tomorrow, false

```

> <i class="fa fa-info-circle"></i>&nbsp;&nbsp; **Extra Note:** 
> If internationalization is a big concern, you should account for unicode characters (ie. david本@abc.com). The regex above works if you first convert your string to [Punycode](https://en.wikipedia.org/wiki/Punycode).

## B. URLs

URL extraction is more complicated because we need to recognize all of the following formats:
```
https://www.google.com
www.google.com
google.xyz
google.xyz/dogs-cats?dogs=500
david.google.xyz:3000/dogs#33
google.xyz?s=i+like+turtles
```
  
And we need to ignore the following formats:
```
bye.done with trial
I am at Google.come to the park.
user@google.com
```

The best regex I’ve seen looks for the pattern <kbd>wx.yz</kbd> where w contains http(s):// and/or www , x contains valid characters for a domain name, y is in a list of valid domain endings, and z belongs to a set of valid characters for url parameters, port, etc.

Here's that regex (make sure to run in case-insensitive mode):
```
(?:https?\:\/\/)?[\w\-\.]+\.(?:'+'|'.join(top_level_domains)+')[\w\-\._~:/\?#\[\]@!\$&%\'\(\)\*\+,;=]*
```

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; The extract_urls function of <a href="https://github.com/davidmace/practicalbots">bot_helpers.py</a> contains code for this task.

And here's my list of common url endings that has 98% coverage:
> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; [top-level-domains.txt](https://en.wikipedia.org/wiki/List_of_Internet_top-level_domains)

Be careful not to recognize emails (david@google.com) as urls (google.com). To solve this problem, first extract emails then check that extracted urls are not substrings of your extracted emails.

> <i class="fa fa-info-circle"></i>&nbsp;&nbsp; **Extra:** 
> I think it's overkill but if you need better than 98% url ending coverage, you can find the updated full list here: [wikipedia top level domains](https://en.wikipedia.org/wiki/List_of_Internet_top-level_domains)

> <i class="fa fa-info-circle"></i>&nbsp;&nbsp; **Extra:** 
> Like emails, urls can contain unicode characters. For instance .рф is the top level domain for ~0.1% of websites. If internationalization is a big concern, the regex above works if you first convert your string to [Punycode](https://en.wikipedia.org/wiki/Punycode).

## C. Phone Numbers
Phone number extraction is more complex than you might imagine. If we want to handle international numbers, we have to consider all of these cases and more :
```
+12-555-555-5555
555 555 5555
5555555
555.5555
555-5555 x.7
555-5555, Ext 8
555-5555 extension 3
(01 55) 5555 5555
0455/55.55.55
05555 555555-55
```

But not these:
```
4500
I have 100. 1000 are on the way
I have between 100-1000 berries
```
Without taking nearby words into account, it’s not possible to differentiate some phone numbers from normal numbers (ie. 5555555 or 100-1000). This is rare enough that I usually just mark any number with seven or more digits/dashes as a phone number.

Here's the regex (make sure to run as case-insensitive):
```
[\d\+\(][\s\-\d\+\(\)\/\.]{5,}[\s\.,]*(?:x|ext|extension)?[\s\.,\d]*[\d\)]
```

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; The extract_phone function of <a href="https://github.com/davidmace/practicalbots">bot_helpers.py</a> contains code for this task.

## D. Numbers

Extracting digit-based numbers is simple. However people often don't write numbers as digits. They say "five thousand" or "5k". Regex isn't the best move since we probably want to map 72 thousand -> 72000 rather than just recognizing that 72 thousand is a number.

Here are some examples of the weird cases we need to handle:

```
9000
9,000
9.0
1.5m
a million
four hundred thousand and twenty eight
72 thousand
one and a half
```

Also make sure to ignore numbers that we have previously identified as parts of phone numbers.

The files below specify written-out number parsing rules (ie. eighty two, 八十二, quatre-vingt-deux) in a format that our parser (process_numbers.py) can process. The examples below are rulesets for English, Mandarin, and French. If you need to support another language, all you have to do is write a new ruleset file in the specified format. The parser will do the rest of the work.

'N' specifies a number word, 'B' specifies a base word, and 'L' specifies a link word. All words after the colon are interchangeable ways of representing the integer or decimal value before the colon.

<br>
<i class="icon-file"></i> **numbers-english.txt**
```
N 0: zero
N 1: one
N 2: two
N 3: three
N 4: four
N 5: five
N 6: six
N 7: seven
N 8: eight
N 9: nine
N 10: ten
N 11: eleven
N 12: twelve
N 13: thirteen
N 14: fourteen
N 15: fifteen
N 16: sixteen
N 17: seventeen
N 18: eighteen
N 19: nineteen
N 20: twenty
N 30: thirty
N 40: forty
N 50: fifty
N 60: sixty
N 70: seventy
N 80: eighty
N 90: ninety
B 100: hundred
B 1000: thousand
B 1000000: million
B 1000000000: billion
B 0.5: halves half
B 0.33: thirds third
B 0.25: quarters fourths quarter fourth
B 0.2: fifths fifth
B 0.1: tenths tenth
L: and_a and
```

<br>
<i class="icon-file"></i> **numbers-mandarin.txt**
```
N 0:零
N 1:一
N 2:二
N 3:三
N 4:四
N 5:五
N 6:六
N 7:七
N 8:八
N 9: 九
B 10: 十
B 100:百
B 1000:千
B 10000:万
N 0.5: 一半 二分之一
N 0.33: 三分之一
N 0.25: 四分之一
N 0.2: 五分之一
N 0.1: 十分之一
```

Here are a few rulesets (English, Mandarin, French):

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; <a href="https://github.com/davidmace/practicalbots">Rulesets</a>

And here is a link to the number parsing code. It handles the language-specific number formats above and non-language-specific number formats (ie. 5.2k, 678).
> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; The extract_numbers function of <a href="https://github.com/davidmace/practicalbots">bot_helpers.py</a> contains code for this task.

<br>
> <i class="fa fa-flag-o"></i>&nbsp;&nbsp; **Limitation:** The number parser cannot handle number formats where the base is on the left of the number (ie. 十分之三 is 3/10 but the first three characters specify the base 1/10 and the last character specifies the number 3). Also because the parser is language-agnostic, it will recognize numbers formats that aren't allowed in some languages (ie. two thousand twenty thirty three-> 2053). I have never seen this cause an issue in practice.


## E. Time Extraction

Times are the hardest entity to properly extract. Ideally we also want to map each time string to a numerical calendar value which we can query later. Here are some examples to show why it's so difficult :

```
yesterday at 7:30am
second Wednesday of April
in 35 minutes
January 4th at 3pm
tomorrow at half past noon
```

Wit.AI’s Duckling has the highest accuracy of any library I've seen. It supports English, Spanish, French, Italian, and Chinese, which together cover 67% of online spoken language. More importantly it's written in a way (probabilistic models) that makes it easily extensible to further languages.

Duckling additionally offers number, email, url, etc parsing; however, I’ve found regex parsers to work better on all entities except time (as of Sept 2016). 

Duckling is written in clojure so I wrapped it in a simple service. Duckling does a non-negligible amount of work so you'll probably want to easily scale it up/down later (which is easiest if it's a separate service). 

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; <a href="https://github.com/davidmace/practicalbots">Duckling Wrapper</a>


## F. People and Locations
Here is an example of extracting names and locations from text.

```
  1. hi im sakthi -> {sakthi:name}
  2. minh is in south dakota -> {minh:name, south_dakota:location}
  3. san jose is better than sf -> {san_jose:location, sf:location}
```

The simplest approach to recognizing names is to make a big list of names from census records. This is possible. However, it’s more scalable to use an established Named Entity Recognizer (NER) because its performance will improve year over year. It’s also hard to get name lists working well for foreign names (ie. John-Paul, Veeral, Vi). There are so many odd possibilities that you’ll probably miss the rarest 10% of cases.

I recommend Google’s NL API Entity Recognizer because it's fast, scalable, decently although not perfectly accurate, and will likely improve over time.

If you don't want to pay or wait for Google's NL API and don't need locations, here is a list of ~5000 person names that performs moderately well. I created it by removing common words (ie. my, april, joy) from a list of the most common names in US census records. US names are pretty thoroughly ethnically distributed.

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; <a href="https://github.com/davidmace/practicalbots">names.txt</a>


# Preprocessing

## A. Replacing Entities with Common Keys

Before classification, we need to perform a few preparation steps.
1. make all words lowercase
2. replace each entity with the capitalized name of its category (ie. 111-111-1111 -> PHONE, david -> NAME).

Now we can more easily recognize entities that appear often within a class. For example now the feature NAME might appear in every example of class:name.


## B. Spelling Correction

Hunspell is a high accuracy, easy to deploy spellchecking solution. It's built into Chrome and Safari among others, so it's likely to continue being supported. It also works in a large variety of languages, which is important for our internationalization goal.

Hunspell overcorrects entities because it doesn't recognize them. This is one reason why it was important to extract these parameters and hide the entity behind a common key before this section. 

Below is a service wrapper for the hunspell open source library so you can access it in any language. Hunspell does a considerable amount of work, which is why it makes sense to deploy it as a separate service (so you can easily scale it up/down later).

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; <a href="https://github.com/davidmace/practicalbots">Hunspell Wrapper</a>


## C. Colloquialism Correction

Spelling correctors don't pick up on Internet slang like <kbd>thx->thanks</kbd> and <kbd>kk->okay</kbd>. For example we might want to respond "No problem" to any form of "thanks" "thks" "thx" "ty". If we had all the data in the world, our algorithm would learn to recognize each of these responses separately. However in most cases we will never see at least one of these forms of "thanks" in the training data. Because of this, we need to normalize the forms.

Here is my list of English internet abbreviations and their full spellings:
> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; <a href="https://github.com/davidmace/practicalbots">Colloquialism List</a>

<br/>
> <i class="fa fa-info-circle"></i>&nbsp;&nbsp; **Extra:** 
I only extracted colloquialisms for English. To generate a colloquialism list for another language, here's a process that works well. Obtain a large amount (1M+ lines) of online conversation data. I used a day's worth of public Tweets. Eliminate every word that is present in a dictionary. Sort the remaining words by frequency. Manually go through the 500 words with highest frequency and extract all of the colloquial words in the list.


# 3. Extracting Custom Entities

## A. Simple Parameter Recognition

What if you want to extract custom parameters? Here are some examples for grocery delivery.
```
i want a bunch of bananas
plz deliver two bags of naval oranges
add a grapefruit
```

We'll worry later about recognizing qualifiers like "two bags" and "bunch". For now we're just trying to recognize singular and plural keywords (ie. oranges, grapefruit, bananas).

This is simple so I won't write out the code. Use the 'stem' function from bot_helpers.py to first reduce plural/singular word forms to a common stemmed form. Then exact match keywords. Remember that multiple words can refer to the same concept (ie. brush, scrubber). 

Also replace each word with the capitalized category name as we did in Section 2A. This will make our later classification task easier.

accountant -> JOB<br>
lawyer -> JOB<br>
doctor -> JOB<br>


> <i class="fa fa-flag-o"></i>&nbsp;&nbsp; **Limitation:** Exact matching parameters isn't perfect, even after spelling correction. It seems at first glance like there might be some nice way to automatically recognize similar words as the same concept (for example by using word2vec distance or wordnet). In practice, these techniques produce far too many false positives and it's surprisingly tractable to quickly write out explicit lists that perform well.

## B. Efficient Spellchecking for Large Lists of Custom Entities

Hunspell only fixes the spelling of common words. If you want to recognize a large (10k+ line) list of proper nouns, you'll need to add more spell-checking logic.

Classic spellchecking algorithms (ie. edit distance) are far too slow if you're trying to recognize huge lists of words/phrases. Here's a clever technique to do quick custom spellchecking on large lists of candidates (source: Popescu et al. Fast and Accurate Misspelling Correction in Large Corpora. EMNLP 2014).

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; The code for this is in the large_list_entity_spellcheck function of <a href="https://github.com/davidmace/practicalbots">bot_helpers.py</a>

The algorithm works by the following logic.

1. Assign each letter/character to a prime number in ascending order (ie. ' ':2 a:3, b:5, c:7, d:11...).
2. Reduce each of your entity names to a number hash by multiplying together the prime numbers that correspond to each letter. (ie. adcc -> 3 * 11 * 7 * 7 -> 1617).
3. When we don't recognize a word/phrase in the user's input, get its number hash by the same method as above.
4. If there is a 1 or 2 character error in the user's entered word/phrase, it will be off from the actual entity's hash by a factor of either 1/a, a, 1/a/b, ab, or a/b where a,b∊[2,3,5,7,11...]. Take all candidate spelling matches with hashes that are off by any of the factors in this list. These are constant-time lookups.
6. The resulting candidate list is vastly reduced so we can do a more expensive edit-distance computation for each of the candidates. This will eliminate candidates that have similar letters to the user's input but in different positions.


# 4. Text Classification

## A. Task and Goals

Text classification is by far the hardest task you'll face when making a bot. For all of the real-world use cases I've seen (sales, customer support, education, insurance, health, news, etc), responding with an incorrect answer is really really bad. Current technology, no matter how hyped, isn't good enough to obtain high accuracy on everything your users will ask. Therefore, our goal is to design a system that can accurately identify what it knows how to answer, pass everything else to a human, and progressively learn from the human answers.

Our goal on a dataset of 2000 samples is to automatically respond to 50% of user inputs with 95% accuracy. The same classification algorithm could probably obtain 70-80% accuracy if we let it respond to all questions, but the 20% of inputs we get wrong will really piss off your customers. As you obtain more data, you should expect to automatically answer more questions while maintaining this consistent 95% accuracy.

## B. Data

Throughout this section, we use a sample real-world dataset from an education provider to test our classifcation algorithm.

For this sample use case, humans manually responded to all user questions until we had 2000 samples. Then we manually grouped the user inputs into classes. Below is a fake but representative example for each class:

**cannot_answer:** can you plz just give me the answer<br>
**blank:** plz<br>
**city:** austin<br>
**fees:** my son is starting school in two days and id like to know how i do payment for him <br>
**course_info:** hello can you tell me more about the underwater basketweaving course <br>
**current_student:** my daughter is an existing student with your school<br>
**dates:** would like to know beginning dates <br>
**domestic:** from the US <br>
**hi:**  hi sir Warm regards  <br>
**international:** not an american citizen <br>
**name:**  billy mcmilly<br>
**new_student:** yup, im junior in hs<br>
**no:**  no thx  <br>
**thanks:** thank you.  <br>
**yes:** yes!<br>
**phone_request:** id like to speak to someone on the phone. this is not working for me  <br>
**email:** botzrkool@hotmail.com  <br>
**phone:** 5555 555 5555  <br>
**assessment:** hi I would like any informaiton on ability assessments. My daughter needs it for her new job. <br>
**stories:** campus life stories from students?  <br>

## C. Results

Before I introduce the model, here are its results on our dataset (70/30 train-test split).

Algorithm answers correctly:  **50.6%**<br>
Algorithm's answer is not perfect but acceptable:  **2.9%**<br>
Algorithm passes question to human. Human cannot label. Respond "sorry I don't know how to answer this.": **17.4%**<br>
Algorithm passes question to human. Human can label: **23.7%**<br>
Algorithm answers incorrectly (due to entity matching error): **2.1%**<br>
Algorithm answers incorrectly (due to classification model): **1.7%**<br>
Algorithm answers incorrectly (no reasonable algorithm would be able to answer): **1.7%**<br>

## D. High Level Algorithm Goals

The goal of our model is to automatically condense a user's input to 1-2 core concepts. Here are a few examples:

```
hi sir can you please help me today in finding a dog -> find_dog
you are not helpful, i would like the support contact number -> contact_number
when can i expect my order will arrive -> order_arrive

```

Text inputs frequently contain garbage words and unnecessary detail. Often the important words are not directly next to each other. Almost always, there are 5+ different wordings of any relationship (ie. order_arrive, delivery_sent, package_arrived). Often far more than five.

Here is a table of the core concepts that I manually extracted from some samples in the 8 hardest classes from my dataset. For 5-15% of samples per class, I wasn't able to extract a core relationship so our algorithm will likely fail on those examples (every other algorithm will likely fail as well). Each column is a class.

{:refdef: style="text-align: center;"}
<img src="/assets/relationships1.png"/>
{: refdef}

{:refdef: style="text-align: center;"}
<img src="/assets/relationships2.png"/>
{: refdef}

It's important to remember that there are ~6-20 possible concepts in an average user input. Our algorithm needs to wade through all that noise to find one or two concepts with valuable signal. This is hard.

## E. Dependency Parse Bigrams

Imagine the user says:
> Can you give me the entire number?"

The relevant piece of information in this sentence is "give_number". This pair is called a bigram.

A common technique for bigram extraction is subsequent words in a sentence: can_you, you_give, give_me, me_the, the_entire, entire_number. Extracting bigrams like this is wicked fast. However these features don't adequately represent distant but related words. The example above shows why this is a problem.

Luckily there is a standard method for extracting more representative bigrams. It's called dependency parsing. Unfortunately the algorithms are slow. However, in my opinion the accuracy gain is worth the speed loss.

Here is an example of a dependency parse (source: Wikipedia):

{:refdef: style="text-align: center;"}
<img src="/assets/dep-parse-img.png"/>
{: refdef}

In the above example, the sentence is transformed into a tree structure. An edge means there's a relationship between two words. To generate bigrams, extract the word pair along each edge in this dependency graph.

For the first example, we now generate the bigrams:
can_give, you_give, me_give, entire_number, number_give.

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; I use Google's NL API to generate the dependency parse then my own code to convert the output to a simple bigram format. That code is in the dep_parse function of <a href="https://github.com/davidmace/practicalbots">bot_helpers.py</a>.

## F. Stemming

Stemming reduces a word to its base form. Here are some examples.

cactus/cacti -> cactus<br>
running/run/ran/runs -> run<br>
cats/cat -> cat<br>

If we map very similar words (car/cars, running/run) to a single concept, we reduce the number of diction forms our algorithm needs to learn. Additionally matching synonyms is much better (I'll write up that code in the future), but just stemming is surprisingly adequate.

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; The code is in the "stem" function of <a href="https://github.com/davidmace/practicalbots">bot_helpers.py</a>.


## G. Short Feature

It's vital that our algorithm doesn't misclassify a long question by recognizing a concept in an unimportant part of the sentence. Here's an example:

Hi -> class:hi<br>
Hi I would like to know what the weather is tomorrow -> class:weather<br>
Hi i was wondering if you offer discounts for residents of California -> class:no_answer<br>

"Hi" is a very significant concept in the first example. It's not a significant concept in the second and third examples. Specifically in the third example, we won't recognize any other important concept to override "hi" since we haven't encountered a request like this before.

This is an extraordinarily hard problem to solve, so we'll use a hack to make due. If the user input is less than 7 words, add a feature 'SHORT' to the list of unigrams and bigrams that represent the input. Now our model can learn to distinguish between simple, easily answerable questions/statements and longer inputs that need more consistent features to properly classify.


## H. When to Pass to a Human

Our model will produce a vector of probabilities that the user's input falls into each category (ie. thanks, hi, name, email, courses, fees, etc).

my name is david -> [0.1, 0.5, 0.1, 0.1, 0.2]

The only nuance is that the first position in the probability vector represents the class cannot_answer. It contains inputs that our human labellers could not classify into a category. Explicitly modelling cannot_answer gives us information on features that might cause high levels of inaccuracy.

We choose the class with highest probability as the response to the input. If this best probability is less than a threshold, we pass the input to a human for manual labelling. Also if the input is classified as cannot_answer, we pass it to a human for manual labelling. Since cannot_answer is such a grab bag of various concepts, we would miss out on a considerable number of answerable inputs if we didn't feed these inputs to a human.


## I. Updating the Model and Verifying Automatic Answer Performance

In general, it's a bad idea to update a model in realtime. Unless your data distribution changes dramatically on a daily basis, the performance gain from realtime updates is miniscule and the potential to crash your model is very high. Instead weekly batch updates are wiser because you can manually verify that your model is performing well before pushing it.

At the end of each week, you have a corresponding response (either from the model or human) for each user input. Append these to your dataset and retrain the model.

There is one small nuance here. If your model gets something wrong (ie. no_problem -> class:no), then you will reinforce that error in your training set. Your model will continue to get it wrong. To resolve this, you should occasionally verify automatic responses after sending them to users. With some probability, P, verify each response by passing the same input to a human. The higher P is, the faster your model will improve and the more labor you'll need. Even with P=0, your model will still improve but it'll continually get the same few cases wrong.


## J. Sifting through the Feature Noise

We need a mechanism to determine which of the thousands of features in our dataset correspond to each class and which ones are noise.

Because we wanted this framework to work on both small and large datasests, most of the work in defining a representative feature (concept) space was done by the feature generation steps above (mostly the dependency parser). These steps aren't trained on our datasets so their performance is agnostic to our dataset size. More complicated statistical models can definitely help refine this feature space by more thoroughly modelling the diction and syntactical relationships. However, it's a much better idea to start with a very simple and interpretable classification model. Then we can switch to a more complex, less interpretable, less simple to maintain model once we've A. built up a very large dataset and B. are confident in our algorithmic pipeline.

Our simple, highly interpretable model of choice is a decision tree.

{:refdef: style="text-align: center;"}
<img src="/assets/decision-tree.png"/>
{: refdef}

Above is part of the decision tree that we trained on our sample dataset. 

TODO explain

> <i class="fa fa-cloud-download"></i>&nbsp;&nbsp; The code is in the "Decision Tree" section of <a href="https://github.com/davidmace/practicalbots">bot_helpers.py</a>.

Each time I build a new model on the training set, I manually look at the trees generated from 3-4 tree sizes. Then you can get a sense of which ones have too few features or too many noise features. You could just choose the model with the best test set performance, but it's nice to have a sense of how your model is developing as it consumes more data.




> Improvements:
1. **Trigrams:** Sometimes bigrams don't suffice to capture full relationships. An example is the relationship "not_existing_student". This problem is actually very rare.
2. **Dependency Graph Transformations:** Sometimes dependency parsing doesn't extract ideal bigrams for feature generation. For example "I would like to run" -> i_would would_like like_run to_run. The most important feature is i_run. There are a few linguistic formats that consistently cause problems like this: i_need_to_verb, i_would_like_to_verb, i_want_to_verb, verb_prep_noun, etc. It might be beneficial to add some feature mapping logic, but I didn't because a core aim of this was to work on multiple languages.
3. **Synonym Combination:** This is the biggest area of improvement for the current algorithm. Reducing the number of unique words our algorithm needs to learn will allow it to properly recognize rare diction.


# 5. State Management

Coming Soon...
<br><br><br><br><br>


