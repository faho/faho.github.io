# cdh - change to a recently visited directory

## Synopsis

```
cdh [ directory ]
```

## Description

`cdh` with no arguments presents a list of recently visited directories. You can then select one of the entries by letter or number. You can also press `Tab` to use the completion pager to select an item from the list. If you give it a single argument it is equivalent to `cd directory`.

Note that the `cd` command limits directory history to the 25 most recently visited directories. The history is stored in the `$dirprev` and `$dirnext` variables which this command manipulates. If you make those universal variables your `cd` history is shared among all fish instances.

## See Also


* the dirh command to print the directory history


* the prevd command to move backward


* the nextd command to move forward