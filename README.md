Twitter Sociolinguistics in the Unix Shell
================

In this note, I'll show how to do a very rough sociolinguistic study of data acquired from the Twitter API, using shell scripting tools only. 
The data comes in JSON format, so I'll focus on parsing that.
If you want to learn more about Unix shell, Ken Church's classic [Unix for Poets](http://www.cs.upc.edu/~padro/Unixforpoets.pdf) is a great way to start.

The sociolinguistic hypothesis that we'll test is that the English future tense system is shifting towards increasing 
use of the less formal "going to" in place of the more formal "will" (see, e.g., [Tagliamonte and D'Arcy 2009](http://www.ling.upenn.edu/~wlabov/L660/Peaks.pdf)). 
According to the sociolinguistic method of *apparent time*, we expect to find that older people remain more likely to use the outgoing form "will", and younger people are more likely to use the incoming form "going to". Let's see whether this holds up in Twitter.

We will use classic shell tools ```grep, sed, awk, cut, sort```, etc. 
I am not a shell expert, and there are likely better ways to do many of these things. 
You can get lots of opinions about these topics on stackexchange and stackoverflow, among other places.

# Looking at the data

Twitter data is stored on our group's server in gzipped text files that contain one [JSON](http://www.json.org/) 
object per line.
If we combine ```zcat``` (```cat``` for zipfiles) with ```head```, 
we can examine the first line of one such file:

```shell
[jeisenstein3@conair new-archive]$ zcat tweets-Feb-14-16-03-35.gz | head -1
","id":58371299,"id_str":"58371299","indices":[3,17]}],"symbols":[],"media":[{"id":696796012514406400,"id_str":"696796012514406400","indices":[36,59],"media_url":"http:\/\/pbs.twimg.com\/media\/CauESBbUcAAyeu-.jpg","media_url_https":"https:\/\/pbs.twimg.com\/media\/CauESBbUcAAyeu-.jpg","url":"https:\/\/t.co\/GhVa5HQ5ok","display_url":"pic.twitter.com\/GhVa5HQ5ok","expanded_url":"http:\/\/twitter.com\/TheybelikeDar\/status\/696796019800014848\/photo\/1","type":"photo","sizes":{"small":{"w":340,"h":628,"resize":"fit"},"large":{"w":554,"h":1024,"resize":"fit"},"thumb":{"w":150,"h":150,"resize":"crop"},"medium":{"w":554,"h":1024,"resize":"fit"}},"source_status_id":696796019800014848,"source_status_id_str":"696796019800014848","source_user_id":58371299,"source_user_id_str":"58371299"}]},"extended_entities":{"media":[{"id":696796012514406400,"id_str":"696796012514406400","indices":[36,59],"media_url":"http:\/\/pbs.twimg.com\/media\/CauESBbUcAAyeu-.jpg","media_url_https":"https:\/\/pbs.twimg.com\/media\/CauESBbUcAAyeu-.jpg","url":"https:\/\/t.co\/GhVa5HQ5ok","display_url":"pic.twitter.com\/GhVa5HQ5ok","expanded_url":"http:\/\/twitter.com\/TheybelikeDar\/status\/696796019800014848\/photo\/1","type":"photo","sizes":{"small":{"w":340,"h":628,"resize":"fit"},"large":{"w":554,"h":1024,"resize":"fit"},"thumb":{"w":150,"h":150,"resize":"crop"},"medium":{"w":554,"h":1024,"resize":"fit"}},"source_status_id":696796019800014848,"source_status_id_str":"696796019800014848","source_user_id":58371299,"source_user_id_str":"58371299"}]},"favorited":false,"retweeted":false,"possibly_sensitive":false,"filter_level":"low","lang":"en","timestamp_ms":"1455353581659"}
```

This is a simple pipeline, which pipes the output of ```zcat``` through the command ```head -1```. (Check out the comment from @WladimirSidorenko on the dangers of using pipes.) Anyway, the structure of the JSON file is:

```{"key1":"value1","key2":"value2",etc}```

However, the values can themselves contain nested JSON objects. Anyway, we want to pull out two fields: "text" and "user.name".

## Using a JSON parser

The easy way to do this is using a json parser, like [jq](https://stedolan.github.io/jq). The following command will pull out the text field from the first three json lines that contain the substring "going to be".

```shell
zgrep 'going to be' tweets-Feb-14-16-03-35.gz  | head -3 | jq .text
```

The result is:

- ```"@StJuliansFC @CwmbranTown hope our games on going to be a nightmare catching up on all these games!"```
- ```"im going to be by myself though im scared"```
- ```"RT @carterreynolds: Magcon may never be the same but we're ALWAYS going to be one big happy family 😊"```

To get the text and the username, you can do this:

```shell
zgrep 'going to be' tweets-Feb-14-16-03-35.gz | head -3 | jq --raw-output '"\(.user.name)\t\(.text)"'
```

- ```machen afc	@StJuliansFC @CwmbranTown hope our games on going to be a nightmare catching up on all these games!```
- ```nicola	im going to be by myself though im scared```
- ```Love yoυ Cαм	RT @carterreynolds: Magcon may never be the same but we're ALWAYS going to be one big happy family 😊```

The format of the ```jq``` command is a little funny: I don't know why you have to escape the opening parentheses, but not the closing ones.

# Getting the text with Sed and Grep

If you don't have ```jq```, or you want to learn to use more general tools, you can use Sed (**s**treaming **ed**itor) to capture each of these fields. (Note that there are command-line JSON parsers, but we're going to stick with the good old linux shell for this tutorial.)

We're going to use Sed's "s" command. The syntax is this command is essentially:

```sed s/find/replace/options```,

where ```find``` specifies a pattern to match and ```replace``` specifies what to replace that pattern with. We'll always use a single option, ```/g``` which says to execute this replacement "globally" -- every time possible in each line.

## One key-value pair per line

As a first step, let's try to break up these big JSON blocks into one key-value pair per line.

```zcat tweets-Feb-14-16-03-35.gz | sed 's/,\"/\n/g' | head -n 5```

Results:

- ```"```
- ```id":58371299```
- ```id_str":"58371299"```
- ```indices":[3,17]}]```
- ```symbols":[]```

Note that this command does not respect JSON's nested structure. For our purposes, that won't matter, but if there were nested "text" fields, we might be in trouble.

## Capture groups

Next, to get the relevant text, we can use a *capture group*. We want to capture the field after the "text" key. Here's how we'll do it:

```sed s/.*\"text\":\"\([^\"]*\)\".*/\1/g"```

This says:

- match all characters before observing the string ```"text":\"``` (note the escaped quotation marks).
- then match a sequence non-quotation characters, ```[^\"]```. The brackets indicate a group (this is a regex), the carrot indicates negation, and then we have the escaped quotation mark.
- by putting (escaped) parens around this pattern, we indicate we want to capture it, ```\([^\"]*\)```
- then match the closing quote, and all other characters in the line
- in the replace string, ```\1``` means print the capture group

We call:

```zcat tweets-Feb-14-16-03-35.gz | sed 's/.*\"text\":\"\([^\"]*\)\".*/\1/g' | grep -E '^[A-Za-z]' | head -3```,

which uses grep to filter the output to make sure it starts with an alphabetic character. Here are the first three results:

- ```https:\/\/t.co\/QuRvL8exJZ```
- ```Que alguien me duerma de una \ud83d\udc4a, \ud83d\ude4f.```
- ```relationship status: https:\/\/t.co\/eON4iSSjvz```

Now let's use a more complicated capture pattern to get the name too. For the name, we will require that it be two, capitalized alphabetic strings, ```[A-Z][a-z]* [A-Z][a-z]*```. This is a trick to trade recall for precision, since there are a lot of garbage names in Twitter. Here's what we run:

```zcat tweets-Feb-14-16-03-35.gz  | sed 's/.*\"text\":\"\([^\"]*\)\",.*name\":\"\([A-Z][a-z]* [A-Z][a-z]*\)\".*/\1\t\2/g' | grep -E '^[A-Za-z].*' | head -3```

Notice that the replace string now is ```\1\t\2```: print the two capture groups, with a tab between them. Here's the output:

- ```https:\/\/t.co\/QuRvL8exJZ	Selena Gomez```
- ```Gusto ko ng katext	Roymar Buenvenida```
- ```RT @NikitaLovebird: \u0915\u0947\u091c\u0930\u0940\u0935\u093e\u0932~ \u0926\u093e\u0926\u0930\u0940 \u091a\u0932\u094b\u0917\u0947 \n\u0911\u091f\u094b\u0935\u093e\u0932\u093e~ \u0939\u093e \u0939\u093e \u0939\u093e \u0939\u093e \n.\n.\n\u0915\u0947\u091c\u0930\u0940\u0935\u093e\u0932~ JNU \u091a\u0932\u094b\u0917\u0947 \n\u0911\u091f\u094b\u0935\u093e\u0932\u093e~ \u0928\u0939\u0940\u0902\n\u0907\u0938\u092e\u0947 \u0915\u0947\u091c\u0930\u0940 \u0905\u0902\u0915\u0932 \u0915\u094d\u092f\u093e \u0915\u0930 \u0938\u0915\u0924\u0947 \u0939\u0948???\n\u2026	Sahil Kataria```

## Putting zgrep in front

We're wasting time processing a lot of text strings that are not of interest. So let's put a grep at the front of the pipeline, so that we start with that:

```zgrep 'going to be ' tweets-Feb-14-16-03-35.gz  | sed 's/.*\"text\":\"\([^\"]*\)\",.*name\":\"\([A-Z][a-z]* [A-Z][a-z]*\)\".*/\1\t\2/g' | grep -E '^[A-Za-z].*' | head -3```

Here are the results:

```- MGBACKTOWEMBLEY	Matt Goss
- God Brat 1, can spew contempt. World champion at ten. He's going to be a fun teenager.	Craig Short
- I've got a new toaster. I've a feeling finding an optimum setting is going to be quite the journey. What a time to be alive.	Boring Tweeter```

Notice that the first example doesn't include "going to be" in the text! The string must appear somewhere else in the JSON, maybe in the profile.

## Postfiltering with grep

The solution is to use grep twice: once as a prefiltering step, and once as a postfiltering step. I find that this is a typical design pattern in streaming from big data: use a fast low-precision filter first, then a slower high-precision filter at the end.

```zgrep 'going to be ' tweets-Feb-14-16-03-35.gz  | sed 's/.*\"text\":\"\([^\"]*\)\",.*name\":\"\([A-Z][a-z]* [A-Z][a-z]*\)\".*/\1\t\2/g' | grep -E '^[A-Za-z].*' | grep 'going to be ' | head -5```

Here are the results:

- ```God Brat 1, can spew contempt. World champion at ten. He's going to be a fun teenager.	Craig Short```
- ```I've got a new toaster. I've a feeling finding an optimum setting is going to be quite the journey. What a time to be alive.	Boring Tweeter```
- ```I've got a new toaster. I've a feeling finding an optimum setting is going to be quite the journey. What a time to be alive.	Boring Tweeter```
- ```I've got a new toaster. I've a feeling finding an optimum setting is going to be quite the journey. What a time to be alive.	Boring Tweeter```
- ```I'm going to be on Broadcasting House on Radio 4 tomorrow explaining why I actually quite enjoy Valentine's Day *ducks*	Henry Jeffreys```

This time I took the first five hits, because the toaster tweet got repeated three times for some reason. We'll deal with that later.

# Collecting names

We only want the names of the individuals using these words. We can collect these using ```cut.```

```zgrep 'going to be ' tweets-Feb-14-16-03-35.gz  | sed 's/.*\"text\":\"\([^\"]*\)\",.*name\":\"\([A-Z][a-z]* [A-Z][a-z]*\)\".*/\1\t\2/g' | grep -E '^[A-Za-z].*' | grep 'going to be ' | head -5 | cut -f 2```

Results:

- ```Craig Short```
- ```Boring Tweeter```
- ```Boring Tweeter```
- ```Boring Tweeter```
- ```Henry Jeffreys```

Let's write these to a file, one for a each pattern:

```zgrep 'going to be ' tweets-Feb-14-16-03-35.gz  | sed 's/.*\"text\":\"\([^\"]*\)\",.*name\":\"\([A-Z][a-z]* [A-Z][a-z]*\)\".*/\1\t\2/g' | grep -E '^[A-Za-z].*' | grep 'going to be ' | cut -f 2 | tee ~/going-to-be-all-names.txt```
```zgrep 'will be ' tweets-Feb-14-16-03-35.gz  | sed 's/.*\"text\":\"\([^\"]*\)\",.*name\":\"\([A-Z][a-z]* [A-Z][a-z]*\)\".*/\1\t\2/g' | grep -E '^[A-Za-z].*' | grep 'will be ' | cut -f 2 | tee ~/will-be-all-names.txt```

Instead of the usual ```>``` redirect, I've used ```tee```, which will print to standard out, as well as write to the file.

## Filtering names

One thing we notice is that one guy, "Cameron Dallas", seems to account for a lot of these messages. Let's just count each name once.

```sort -u ~/will-be-all-names.txt | cut -f 1 -d\  | uniq -c | tee will-be-name-counts.txt```

Here's what's happening in this pipeline:

- ```sort -u``` alphabetically sorts all names, and returns the unique entries
- ```cut -f 1 -d\  ``` selects only the first name, by cutting on a single whitespace delimiter. This is valid because we are only admitting names that contains exactly two whitespace-delimited tokens.
- ```uniq -c ``` computes the *count* of all unique entries. ```uniq``` requires that its input is already sorted, but we have this from the ```sort``` call at the beginning of the pipeline.

# Looking up ages per name

Let's get the top names for each set:

```sort -nr ~/will-be-name-counts.txt  | head -5```

Results:

- ```     10 The```
- ```     10 David```
- ```      8 Michael```
- ```      8 Chris```
- ```      7 Mark```

```sort -nr ~/going-to-be-name-counts.txt | head -5```

Results:

- ```      3 Taylor```
- ```      3 Nathan```
- ```      3 Josh```
- ```      3 Jon```
- ```      3 Hannah```

Whose are older? Some answers are online: [http://rhiever.github.io/name-age-calculator/index.html?Gender=M&Name=Nathan](http://rhiever.github.io/name-age-calculator/index.html?Gender=M&Name=Nathan)

## Pulling web data

We can also pull data directly from this site:

```curl http://rhiever.github.io/name-age-calculator/names/M/N/Nathan.txt```

This data is field-delimited by a comma. We will use ```awk``` to reorganize it so that the column containing the counts of living people is first. Then we will sort on this column, to find the year in which the greatest number of living people was born with name:

```curl http://rhiever.github.io/name-age-calculator/names/M/N/Nathan.txt | awk -F, '{ print $3"\t" $1 }' | sort -n | cut -f 2 | tail -n 1```

Results: ```2004```

The ```awk``` command ```awk -F, '{ print $3"\t"$1 }'``` says:

- Use the comma as a field delimiter
- Print the third field, then a tab, then the first field

## Pulling data for each name

We want to get this data for all names. Let's see how we can iterate over the lines in a file. We'll use the ```for``` command, and then use $(command) to indicate a variable whose values are defined by the output of a sequence of other unix commands:

```for name in $(sort -nr ~/will-be-name-counts.txt | head -5 | sed 's/\([^[:alpha:]]*\)//g');  do echo $name; done```

Next, we'll take these names, and dynamically construct appropriate URLs to pull their age statistics from the from the website. Then we'll run our awk postprocessing script to get the ages.

```for name in $(sort -nr ~/will-be-name-counts.txt | head -5 | sed 's/\([^[:alpha:]]*\)//g');  do echo $name; curl -s http://rhiever.github.io/name-age-calculator/names/M/${name:0:1}/$name.txt | awk -F, '{ print $3"\t"$1 }' | sort -n | cut -f 2 | tail -1; done```

Note that the URLs require us to specify male or female. The above command line only gets male results. Let's add another ```for``` loop to iterate over genders. (@WladimirSidorenko notes that the iteration style ```for gender in {'M','F'}``` was only supported since Bash 4.0, so you may prefer the "old school" alternative ```for gender in M F```.)

```for name in $(sort -nr ~/will-be-name-counts.txt | head -5 | sed 's/\([^[:alpha:]]*\)//g'); do echo $name; for gender in {'M','F'}; do curl -s http://rhiever.github.io/name-age-calculator/names/$gender/${name:0:1}/$name.txt | awk -F, '{ print $3"\t"$1 }' | sort -n | cut -f 2 | tail -1; done; done```

Surprisingly enough, most of these names have significant counts for both girls and boys. But here are the final results:

- The
-      ul { list-style: none; margin: 25px 0; padding: 0; }
-      ul { list-style: none; margin: 25px 0; padding: 0; }
- David
- 1960
- 1983
- Michael
- 1970
- 1986
- Chris
- 1961
- 1961
- Mark
- 1960
- 1968

We get an error for the non-name "The", which returns an HTML string. For the names that work, most of them are for people born in the 1960s. Now for "going to be":

```for name in $(sort -nr ~/going-to-be-name-counts.txt | head -5 | sed 's/\([^[:alpha:]]*\)//g'); do echo $name; for gender in {'M','F'}; do curl -s http://rhiever.github.io/name-age-calculator/names/$gender/${name:0:1}/$name.txt | awk -F, '{ print $3"\t"$1 }' | sort -n | cut -f 2 | tail -1; done; done```

- Taylor
- 1992
- 1993
- Nathan
- 2004
- 1985
- Josh
- 1979
- ul { list-style: none; margin: 25px 0; padding: 0; }
- Jon
- 1964
- 1958
- Hannah
- 2004
- 2000

Taylor, Nathan, and Hannah all peaked after 1990; Josh peaked in the late 1970s, and only Jon dates back to the 1960s.

This data, limited though it is, supports the hypothesis: "going to be" is favored by younger writers on Twitter.

# Next steps

This analysis is limited and flawed in many ways. 
To make it more robust, I might *first* extract all relevant tweets, which are those that contain either "will be" or "going to be". 
Then I could work with this smaller dataset, which will be faster to analyze. 
For each author, we could try to impute an age by the name, and then use logistic regression to see how much of the variance is explained by author age. 
I'd want to do that in Python statsmodels or R. To get there, I would likely use shell or python to write out CSV files which could easily be read into dataframes.
