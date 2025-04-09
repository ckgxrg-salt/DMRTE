# DMRTE  
Rust-flavoured DMIAE.   
DMIAE is a protocol to represent stories or scripts, for use with games of plays.   
This repository contains DMRTE, a tool that converts plain text into DMIAE format.   

# Table of Contents
- [Structure](#structure)
- [Source Script File Syntax](#source-script-file-syntax)
- [Source Script File Properties](#source-script-file-properties)
- [Script Data Structure](#script-data-structure)
- [Tips and Tricks](#tips-and-tricks)

# Structure
To use DMIAE, a source script file is needed.   
The source script file should be in plain text format, and will need to be wrote in the syntax defined below.   
After using DMIAE to parse the content, a file containing the serialised result will be generated.   
This generated file can then be used with DMIAE API to extract contents.   
In short, DMIAE API defines the data structure for a script; DMRTE CLI, or any other applications using the API, manipulates the data freely.   

# Source Script File Syntax
This section describes how DMRTE's parsing process works.   

## Tags & Paragraphs
DMIAE uses tags and paragraphs to group lines together.   
Tags can be applied by using a special syntax.   
Tags are primarily aimed to better search in the script. the DMRTE CLI allows displaying all lines with certain tags.   

Paragraphs represent a grouping of texts. Usual scripts contain something like "Act III" or "Scene IX", and "Paragraphs" are just like that.   
Paragraphs can be nested, so instead of "Act I Scene II" you can write "Act I" followed by "Scene II".   
Paragraphs are the basic structure of DMIAE. Actually, a DMIAE "Script" is just a combination of "Paragraphs": Characters' Lines are put inside their respective paragraphs(no free lines are allowed).   

## Types of lines
- Comments
- Properties
- Character Declarations
- Paragraph Marks
- Characters' Lines

## Comments
```
// [comment]
```
Lines starting with "//" will be treated as comments.   
The source file is aimed to be human-readable. For information like name or author, writing them in comments is a good way.   
DMRTE discards any comments in the script.   

## Properties
Lines starting with "!" will be treated as properties.   
Each property's syntax is somehow different, see them in the next section.   
Properties are either static or dynamic.   
Static properties provide some information about the script and should not be defined more than once. The DMRTE CLI disallows stating a static property twice. Location of a static property does not matter.   
Dynamic properties provide additional information over actors' lines. Defining them multiple times have different effects, see their respective descriptions. Usually, dynamic properties affect lines after them.   

## Character Declarations
Lines with this format declare a character:
```
@<character name>, [alias], [alias]...[:description]
```
This line must be defined outside any paragraph, so most naturally in the beginning, though other location is acceptable.   

At least one name of the character have to be declared, this will be the identifier of the character and will be internally used by DMIAE API.   
Colons can be used after character names to write descriptions.   
For characters with multiple aliases, use commas to separate each alias.

Character names may contain spaces, but spaces around the name will be ignored.   

Any characters that appears in the script must be declared, otherwise they'll be identified as part of other characters' line.   
Character names that declared in the header should be **EXACTLY THE SAME** as they appear in the script.   
```
@Elle, Legally Blonde
@Hamilton
@Judge: A maniac
```

## Paragraph Marks
">" starts an paragraph block.   
```
> <paragraph name>, [alias], [alias]... [:description]
```
Paragraphs, like characters, can also have alias, mostly used in multilingual situation.   
Use "<" to terminate an paragraph, it will only terminate the innermost paragraph.   
These marks can appear multiple times in a line, so `> Act I > Scene II` is acceptable.   
A parent paragraph cannot have a child paragraph with the same name, so if you write this:
```
> Act I > Scene I
<
> Act I > Scene II
<
```
The result will be:
```
Act I
  - Scene I
  - Scene II
```
```
> Act I > Scene I
lines and lines
<
> Scene II
still lines
<<
> Act II
<
```

## Characters' Lines
Character's lines can only appear in paragraphs.   
A colon is used to separate the character name with actual text.   
A line can be defined without a character name and the colon, in the case the line will be seen as the last character's.   
```
Elle: I feel so much better,
I feel so much better,
I feel so much better than before

Emmett: Don't spend hours on my hair
Elle: (I don't spend hours)
```   

### Escape Characters
If you really need to use "//" "#" "@" and such to start a line, use "\" to escape them.   

### Multiple Characters
If a line belongs to multiple characters, use slashes(/) to separate them.   
```
HAMILTON/MULLIGAN /LAURENS/LAFAYETTE
I am not throwing away my shot.
```   

### Tags
Use this syntax to tag a line:
```
#<tag name> [character name:] <content>
```
Thus, whitespaces are not allowed in tag names.   

Tags can be used to categorise lines, for example, if you have a bilingual script:
```
#en William: Catch me if you can.
#cn 威廉: 能的话就来抓我吧.
```
Then you can form full script in a specific language using the tag.   
However, manually define these tags for each line is boring, so the "tagrule" dynamic property can help you to automatically tag lines according to a certain format.   

Tags can be removed by prepending the tag name with "-", so `#-en` removes the "en" tag from a line.   

Tags are usually preserved in the generated DMIAE file, but tags that start with `#!dmrte.`(e.g. `#!dmrte.tagrule.ignore`) will be discarded by DMRTE (To be precise, these tags are consumed by DMRTE in the generation process and there's no need to save them).   

### Actions
Actions are performance not delivered by words, like light，music，or just notes.   
Use this syntax to define Actions:
```
[character name:] <content (@[action type:] <action content>) still content (@action) also content>
```
Actions are wrapped in lines, and will record their position.   
Actions may also be wrapped in empty lines.   
If no action type was specified(that is, no colons), action type will be set to "".   

An Action has a type and content, type is any given string.   
Lines with actions automatically gain the "#action-type" tag.   

Example:
```
(@Light: 红1面2)
All: Blood in the water.(@Note: 开始逼近)
水中之血。
Callahan: Becomes your only law.
成为你唯一的法则。
(@Callahan关门)(@Audio: Blood in the water结束)
```   

# Source Script File Properties
Not all properties will by default be recorded into the parsed file. Static properties are always recorded, while dynamic properties' situations vary.

## Static Properties

## Dynamic Properties
### tagrule
"tagrule" defines a format to automatically tag lines.   
Discarded by default.   
```
!tagrule [nomark] <relative position> [tag] [+tag] [-tag]...
!tagrule reset
```
Where relative position is any of:
- this
- after <number of lines>

tagrule applies to all following lines.   
For each line, DMRTE will manipulate tags according to the relative position.   
Lines affected by tagrule automatically gains `#!dmrte.tagrule.ignore`, which prevents the line being treated as "this" in tagrule again, this behaviour can be disabled by adding [nomark].   
Lines that has no content(i.e. nothing is "spoke" by anyone, like a line with only actions) will also gain `#!dmrte.tagrule.ignore`.   
Also, manually adding this tag prevents tagrule from applying to this line.
```
!tagrule this +en
!tagrule after 1 +cn

Whiteley: How did you...? <- Let's say DMRTE is looking at this line
怀特利: 你怎么...?

Whiteley: How did you...? <- referred as "this"
怀特利: 你怎么...? <- referred as "after 1"

Whiteley: How did you...? <- add tag "#en" and "#!dmrte.tagrule.ignore"
怀特利: 你怎么...? <- add tag "#cn" and "#!dmrte.tagrule.ignore"

Whiteley: How did you...? <- 
怀特利: 你怎么...? <- DMRTE attempts to move forward, but this line already has "#!dmrte.tagrule.ignore", so it will move forware again.   
```
[+tag] or [tag] adds a tag to the line, [-tag] removes an existing tag.   
Redefining tagrule adds a tagrule, use `!tagrule reset` to clear all tagrules.   

# Script Data Structure
This section describes the internal data structure of a DMIAE script.   

DMIAE Script
  - Static Properties
    - property
    - property
    - ...other static properties

  - Characters
    - character 1
      - aliases
      - description
    - character 2...

  - Some Paragraph
    - aliases
    - description
    - parent
    - line 1...
    - Another Paragraph
      - line 1
        - character
        - text
        - tags
        - actions
          - type
          - content
      - line 2...

## Static Properties
A struct of nullable fields containing static properties.   

## Characters
HashMap with identifier as key indexing Character structs.   

Each Character struct contains their identifier, an optional list of aliases, and an optional description.   

## Paragraphs
Main contents.   
HashMap with identifier as key indexing Paragraph structs.   

Each Paragraph struct contains their identifier, a nullable reference to its parent, an optional list of aliases, an optional description, an optional list of Line structs and an optional list of child Paragraphs.

Each Line struct contains its belonging character, the text, an optional list of tags, and an optional list of Action structs.   

Each Action struct contains its type and an optional description.   

# Tips and Tricks
## Use multiple source scripts
This is useful when you are working on a multilingual project and don't want to mess up your contents.   

### Characters & Paragraphs
Character declarations, as long as the "identifier" is the same, can be defined multiple times to add aliases to a character, for example, you have 2 versions of a script, an English one and a Chinese one:
```English version
@William
```
```Chinese version
@William, 威廉
```
And DMRTE will recognise these to be the same character, besides using a specific language, you can also use something else to "anchor" these characters.   

Paragraphs also support alias, so they are basically the same.   

### Contents
DMRTE will combine Paragraphs with the same name.   
You have to specify how DMRTE arranges the combined lines.   
If you don't care about line numbers, you may simply "append" one version to another version.   
Or if you are doing translations, you can insert the lines "one-after-one".   
