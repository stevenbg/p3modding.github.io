# Charge
Charge letters are [messages](../letters.md) of type `0x0A`, sent by
[criminal investigation](../scheduled-tasks/0005-criminal-investigation.md) scheduled
tasks. The task writes the type byte at `0x004E5CEB` before handing the message to
`add_message`.

## Text
A charge letter always starts with *Today, you have been charged for the following reason*.
The following text depends on the crime type:

|Crime Type|Text|
|-|-|
|5|*You have been seen behaving indecently.*|
|6|*You are accused of heresy.*|
|7|*You have stated that the world is round.*|
|8|*You stand accused of taking part in a punishable offence, undermining the good of the Hanseatic League.*|

Finally the suffix *We will thoroughly examine this charge and inform you of the outcome of investigations as soon as possible* is appended to every accusation letter.
