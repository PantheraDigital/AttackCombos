# Implementing Attack Chains/Combos
### Problem
Attacks need to be linked together in a way that successive attacks are different based on context and/or input. 

Input being the attack types related to buttons the player may push (light, heavy, etc.).
Context being the state of the character (attacking, dodging, holding a weapon, in air, etc.).

### Solution
With two main components this can be implemented and support different levels of complexity. First the container will hold the data related to the attacks. Then a queue will aid in the order attacks are played and provide extra context for future attacks.

## Container
The container’s goal is to simply hold attack animations and return the correct one when requested based on input from an external class (such as the character controller). The container will internally handle tracking which attack was used last and select the next attack from there.

### Linear Attacks
![LinearArray drawio](https://github.com/user-attachments/assets/c7592692-5dd5-43ff-874c-cd869921f210)

Use an array. Store the index of the active attack and increment the index as attacks are performed. Reset the index to 0 when the end is reached, or set to 0 after a short delay to reset the chain after a lack of input so that the chain does not start in the middle after an extended period of no input/non-attack inputs.

To increase attack variety multiple arrays may be used and swapped as the character equips different weapons, changes states (such as being in the air), or if the player provides a different input for a different type of attack.

![LinearCombo](https://github.com/user-attachments/assets/ca28b700-172d-4828-9caa-ca636e81e801)

The container would track the currently selected array and current index.

The interface for this container would look like this:

`GetNextAttack(attackType)`
> Returns the next attack based on the current index and currently selected array. This is also where different arrays will be chosen if the next attack in the currently selected array does not match the requested attackType. The current index is also incremented here and set to 0 if the end of an array is met.

`Reset()`
> Set the current index to 0.

This method is good when not many attacks are used or when other variables do not greatly impact the possible attacks output by the character.\
The greatest limiting factor is the re-usability of attacks since they are stored in linear paths, meaning every possible combo needs to be defined from start to end in its own array. Depending on the game this could quickly get out of hand.

![MultiLinearArray drawio](https://github.com/user-attachments/assets/2da048fc-a023-4499-9ef8-1349ced8ea74)

Above are four arrays defining different chains of attacks the player may use. This shows the problem of complexity as many arrays will need to be defined to allow different attack chains (more will be needed to cover the different states of the character like aerial attacks or attacks after dodges). More importantly, however, this shows the glaring issue of how do you choose the right array to use based on a single input since the future inputs of the player are unknown and may be chaotic.

### Diverging Path Attacks
Use a tree to store the attacks. Each node holds the attack animation and any other needed data.\
Attacks can follow diverging paths based on input or other variables.

![BranchingPath1 drawio](https://github.com/user-attachments/assets/6a982c2b-3274-4099-9a9c-9148f4f530f6)

Start = no specific character state (idle, movement, dodge,...). Holds no animation. \
Light attack + light attack + light attack = 1,2,3 \
Light attack + light attack + heavy attack = 1,2,4 \
Heavy attack + heavy attack = 5,6

With each input the attacks progress down the tree till the end of the chain. \
Once at the end the active node will wrap back to the beginning and start over, following the path of the input. (mashing light attack will loop 1,2,3,1,2,3… in the above tree.) \
Nodes hold unique attack animations, or duplicate ones. As the player inputs attack inputs of different types (light or heavy) the tree progresses to the next node so that attack can be played.

Link nodes to each other to set the different “paths” players may progress through as their input changes. 

![BranchingCombo](https://github.com/user-attachments/assets/f6c48f58-d565-46e4-a8d0-98a948bee108)

#### How do you determine which node to move to?
Use a formatted key that represents the data needed to choose an attack, where every part of the key represents a different variable related to choosing an attack. 

For example: \
If your game is only concerned with an input attack type and a directional input given when the attack input is given. \
(The desired effect with this would be that holding forward and performing a light attack would be different from holding back, or no direction, and performing a light attack.)

Strict format: {direction}{attack}
 - Keeps keys consistent
 - Easier to check
 - Establishes a fixed size

Do not allow missing variables, use stand in characters if the variable is null or ignorable in specific cases. Each variable should occupy a set amount of character spaces and always be in the same position.

An example key would be “0,1L” (forward light attack). When checking this key we know we can extract the first three characters for the direction and the last character for the attack type. 

Here is how a tree might look with its keys:

![BranchingPath2 drawio](https://github.com/user-attachments/assets/ad7992a5-25cd-4c88-9fd7-89386824c9a0)

Gather input then translate it to the key format. (Here direction is a 2d normalized vector, L is light attack, and H is heavy attack)

From the current node, select the best matching key. The keyword here is “best”, and this will depend on your situation, because the goal is to always return something or else no attack can be played for an input and the player will be left thinking something is wrong with the game. Using the above tree you can see the player does not have access to a heavy attack unless they input a light attack first. 

![BranchingPath3 drawio](https://github.com/user-attachments/assets/4f4be0a0-7677-4003-8f3a-a0d01a14f5af)

If the orange node is the current attack and the next key from the player is 0,1H what node do you return? There is no match for this key, which is why determining your “closets match” logic is important. 

If no direct match is found, find the closest match, if nothing is close enough then loop back to the start and try again.

“Close enough” is determined by what variables in the key can be changed and which should not be. \
The attack type should not be changed (a player should never perform a heavy attack when a light is input). The move direction is adjustable however (if the player is holding a direction it would be ok to return an attack associated with no direction input if it means being able to return something). \
So in the case of the orange node and a key of 0,1H the next node selected would be 0,0H. This is also because we don't want to totally change the direction of movement. You don't want the player to do a backwards attack when they are holding forwards. 

```
Check for match
If no match
    Set movement to 0,0
    Check for match
    If no match
        Check for match from start using original key
        If no match
            Set movement to 0,0
            Check for match from start
            If no match
                Return null // failed to find an attack that is “close enough”
```

This also demonstrates a need to fill out the tree in a way so that if the chain is at the start there will always be a match, or risk missing an input. 

#### How do you catch all inputs without bloating the tree?
Special characters can be used as internal tools in the tree container that allow for an easy way to catch all input by signalling the matching logic to use special logic.

Special characters:
 - Maintain required key size
 - Use characters not normally used
 - Allows for custom logic in special cases

Since there is a strict format this lets us use keys like &,&L and know that the first three characters are for the movement and the last is the attack type. By using ‘&’ instead of numbers we signal to match this direction with any direction.

This will make the key match with any light attack key since the direction can match any direction. Because of this the key matching logic needs to account for this and match with special characters last, or risk them matching before a real match or close match.

```
Check for match
If no match
    Set movement to 0,0
    Check for match
    If no match
        Set movement to &,& // check for any key with matching attack type
            Check for match
            If no match
                Check for match from start using original key
                If no match
                    Set movement to 0,0
                    Check for match from start
                    If no match
                        Set movement to &,&
                        Check for match from start
                        If no match
                            Return null // failed to find close enough match
```

Now with special characters the tree can include nodes like this and catch any key input and prevent missed inputs.

![BranchingPath4 drawio](https://github.com/user-attachments/assets/cea7bc67-4aa1-4e23-9607-d0b1311ffbbd)

Be careful adding nodes with a key like &,&& as this would catch ANY input keys without a match. I advise only using special characters for variables in the key that get changed in the matching logic. 

By using &,&H and &,&L different attacks can still be output matching the attack type, rather than a single attack for both light and heavy. However, how you cover your bases is up to you and whatever fits your game. You may even not use special characters and have a key linked to the start for every possible key. 

#### How can this be expanded for more variables?
Expanding this to cover more variables can be done by adding them to your key format and making sure your matching logic accounts for this. 

Let's add a “previous action” variable to the key. The previous action is an action performed before the attack that is not an attack (dodge, parry, block). These actions are only combat related actions that will change the output attack (not walking, idling, or similar). \
(The desired effect here is to allow custom attacks after performing a forward dodge or a special attack after a successful parry.)

First update the key format. \
{prev. Action}{move direction}{attack type} \
 - Same but with the added variable at the beginning

Establish variable characters. \
Previous action characters: \
 - P (parry)
 - D (dodge)
 - B (block)
 - ! (none, special character)
   - Used when a different action was performed that has no effect on choosing the next attack

Next update the logic to use the new key size and characters. In this case “previous action” is changeable like movement, as in it will not match other actions but changes to a null state.

```
Check for match
If no match
    Set movement to 0,0
    Check for match
    If no match
        Go to start
        Check for match with original key
        If no match
            Set movement to 0,0
            Check for match
            If no match
                Set movement to original
                Set prev. action to !
                Check for match
                If no match
                    Set movement 0,0
                    Check for match
                    If no match
                        Set movement &,&
                        Check for match
                        If no match
                            Return null
```

Now the logic has changed to make adjustments including previous action. (This logic attempts to preserve the action over preserving movement by not changing the action till moving to the start.)

A simple example tree may now look like this:

![BranchingPath5 drawio](https://github.com/user-attachments/assets/dc55d9e8-ac84-425f-a967-7b4a2ade65bc)

The tree now includes a two attack heavy chain that can be performed after a block.
 - B&,&H : heavy attack after block with any movement
 - !0,0H : heavy attack with no movement

It also now includes a chain that includes a dodge halfway through.
 - &&,&H : heavy attack after any action with any movement
	- D0,1H : heavy attack after a dodge with forward movement input
   - This can indicate a forward dodge or movement after the dodge, exact implementation is up to you
   - The dodge also indicates that a dodge was performed between this attack and the last attack in the chain since the variable indicates previous actions

The use of ‘!’ in the chain at the top of the tree indicates no actions are performed during the chain, or else the chain is broken and sent back to the start. ‘&’ may be used if any action can be performed during a chain without breaking it. 

#### Can’t use a tree, or don’t want to?
This may be the case in some environments that make it more difficult to set up a tree, or support the use of a tree, such as in Unreal Engine when wanting to support the Blueprint system for edits to the container. 

Just place the nodes in a key-value paired structure like a dictionary/map. 
 - Use the key as the key and the data as the value. 
 - Chain the keys together with a separator character.

![BranchingPath3 drawio](https://github.com/user-attachments/assets/b9494fa3-aebc-4dd3-a0cf-8601fd347e7f)

To now access the orange node the key would be 0,0L_0,0L where ‘_’ is the separator. \
(Another chain would be 0,1L_0,1H)

All the previous key rules still apply but now a single key includes the leading attacks so that the structure still functions the same. (Key checks will need to account for this by not changing the leading keys, only the end key.)

Key management will need to be implemented to save and reset the storage of the leading keys. \
Instead of just setting a new active node and checking the connected nodes this needs to be simulated by caching the previous keys then appending the next key before a search.

Input: 0,0L \
Cache: empty \
Search: 0,0L

Input: 1,0L \
Cache: 0,0L \
Search: 0,0L_1,0L

(Closest match logic does not find key 0,0L_1,0L but finds 0,0L_0,0L so that is saved to cache.)

Input: 1,0H \
Cache: 0,0L_0,0L \
Search: 0,0L_0,0L_1,0H

Search logic will also need to be adjusted to factor in the now appended previous keys. 

Check for previous keys + separator + input key \
If no match: Loop check for previous key + separator + adjusted input key \
If no match: Check for input key \
If no match: Check for adjusted input key

(Adjusted input key is the input key with adjustments made when finding the closest match.) \
(Separator is the chosen character to separate keys in a chain. Above ‘_’ is used)

### Summary
 - Store attacks and their data with a key associated to them
 - Use the key to represent the context and/or input needed to play that specific attack
 - Keep a strict key format to keep keys consistent and easy to dissect for comparison 
 - Implement logic to find a “closest match” to prevent a null output 
 - Use special characters for special situations related to each variable in the key
   - If a variable is optional a character still news to occupy that space
   - Internally signal the key may match any value in that position

![AttackContainer drawio(1)](https://github.com/user-attachments/assets/b9b3633f-3d47-4856-b101-b588f5994add)

## Action Queue
The Action Queue is a data structure that serves to take in actions from the player as they are input, then output them in the same order, when they are allowed. This is to prevent any actions from being skipped or cut short when they should not be. The queue also allows the parent object to view the different actions in the queue based on sections (“queued” - ready to be performed, “current” - the action being performed, and “previous” - actions that have been performed) so that they may be used when determining the context of new actions. 

Actions being “allowed” to play will be determined by game logic but as an example actions may be allowed to play if the current action’s animation is considered done. This logic will be implemented in the parent object and is out of the scope for the queue. The queue is simply a tool for the parent object to manage the character’s actions.

As an example, if the player inputs an attack it will be added to the “queued” section. The parent object will see that there is no current action being played and immediately progress the queue, placing the attack in the “current” slot and play it. The player then inputs another attack while the current one is playing. This attack is added to the queue, but the parent object sees the current action is not done so it waits. When the current attack finishes the parent object sees an action is in the queue and progresses the queue, moving the first attack to the “previous” section and the second to the “current” slot. Then it plays the new current action. This prevents inputs from being ignored and forcing the player to wait for every action to finish before inputting a new action. 

![ActionQueue drawio](https://github.com/user-attachments/assets/3056b36d-e87c-4942-8c95-ebd3555e9376)

### How Does It Work?
The action queue is a rather straight forward queue data structure that acts as the name implies, with some modified behavior to keep the queue a fixed size along with the addition of sections within the queue that signify the state of an action within the queue. The “sections”/”states of an action” in the queue are “active”, “queued”, and “previous”. 

Only a single action can be “active” at any given time. This indicates the action is the one currently in use by the character and is probably being played on the character. “Queued” actions are next in line to be active while “previous” actions have already been active.

“preserveSize” sets the size of the “previous” section while “queueSize” sets the size of the “queued” section. “arraySize” and “actionArray” are the entire Action Queue. 

Queue progression would look like this:

![ActionQueueExample1](https://github.com/user-attachments/assets/ffcced0f-4144-48ff-8e85-6bc19ef3ca57)

Items in the queue do not move, rather an index is incremented that indicates the “active” action. Using the active index the other sections of the queue can be calculated simply using the size of those sections.

A critical aspect of the Action Queue must be remembered. The queue is a fixed size and will stay this size as it is used, unless specifically resized using a special function. As the queue progresses and as actions are queued, the size will not change. Instead, indexes will wrap back to the beginning when they leave the bounds of the queue and old actions will be overridden.

So using the fact that the queue is fixed size, the index range of the “previous” section and the “queued” section can be represented as such:

`Previous index range = [active index - 1, active index - “previous” size]` \
`Queue index range = [active index + 1, active index + “queue” size]`

The size of the array that acts as the Action Queue would be the sum of its parts. \
`Array size = “queued” size + 1 + “previous” size`

### Keeping Indexes In Range
Ignoring the different sections of the queue for now let's focus on the “active” slot in the queue. As the queue progresses the index of this slot increases to point to a new “active” action. Eventually this will reach the end of the queue and go out of bounds if not handled properly. 

Using the modulus operator (%), indexes can be kept within the queue as modulus returns the remainder of a division. This just so happens to correspond perfectly with the range of indexes in an array.

So the formula to keep an index in the queue is: \
newIndex = indexToCheck % queueSize

This is division based so order is important. The size of the queue should be after the index to check. Extra precautions should also be taken to avoid dividing by 0 and properly calculate negative numbers.

```
FitIndex(index, queueLen)
    If queueLen == 0
        Return -1
    If index < 0 
        Return queueLen - abs(index % queueLen)
    if index >= queueLen
        Return index % arrayLen
    Return index

Int QSize = 3
Int activeIndex = 2
Int usableIndex = FitIndex(activeIndex+1, QSize) // output: 0
```

Now to reassess the previously defined ranges of the “queued” and “previous” sections. They are found relative to the “active” index, which means they are highly likely to go out of range. So to fix this we can simply wrap them back to the start.

`Previous index range = [FitIndex(active index - 1), FitIndex(active index - previous size)]` \
`Queue index range = [FitIndex(active index + 1), FitIndex(active index + queue size)]`

![ActionQueueExample2](https://github.com/user-attachments/assets/7cff3fd4-e170-49d2-ab94-cb70e4684387)

Now the indexes of the actions in the sections can be found by wrapping the indexes that go out of range.

### Action Refs
Action Queue holds three public variables referred to as “action refs”. These “action refs” are `previousAction`, `activeAction`, and `queuedAction`. They are public copies to allow for easy access to these variables. 

`previousAction` is the most recent action added to the “previous” section, and `queuedAction` is the next action to become “active”.

```
UpdateActionRefs()
    If preserveSize > 0
        previousAction = actionArray[FitIndex(activeIndex - 1, actionArray.length)].action
    Else 
        previousAction = Action()
    activeAction = actionArray[activeIndex].action
    If queueSize > 0 and lastQueuedIndex > -1
        queuedAction = actionArray[FitIndex(activeIndex + 1, actionArray.length)].action
    Else
        queuedAction = Action()
```

### Is An Action In A Section?
Another useful function is to find if an index is in a section (“queued” or “previous”). A section is simply a set of indexes within the queue. So to find if an index is in a section we can use the range of the section.

```
// start = the starting index of the range within the queue
// end = the ending index of the range within the queue

IndexInRange(index, start, end)
    Index = FitIndex(index)
    start = FitIndex(start)
    end = FitIndex(end)
    If start == end
        Return (index == start)
    If start < end
        Return (index >= start) and (index <= end) // if index is inside start to end index
    Else
        Return (index >= start) or (index <= end) // if index is outside start to end index

bool indexInQueue = IndexInRange(FitIndex(index), FitIndex(activeIndex + 1), FitIndex(activeIndex + queueSize))
```

This method takes advantage of the fact that all indexes are within the queue. Due to this fact, if the starting index is greater than the ending index, then the range wraps around the queue.

A more generic version of this method would be as follows.

```
// find if an index is within a range of indexes, in an array
IndexInRange(index, rangeStart, rangeLen, arrayLen)
    If (rangeStart + rangeLen - 1) < (arrayLen - 1) // if the range does not wrap
        Return (index >= rangeStart AND index <= rangeStart + rangeLen - 1)
    Else
        Return (index >= rangeStart AND index < arrayLen - 1) OR
            (index <= (rangeStart + rangeLen - 1) % arrayLen AND index >= 0)

bool itemInQueue = IndexInRange(itemIndex, activeIndex+1, queueSize, arraySize);

// itemIndex = the index of the item we are testing
// activeIndex+1 = the starting index of the queue section
// queueSize = the length of the queue section
// arraySize = the length of the action queue
```

### Adding Actions To The Queue
```
QueueAction(Action action, bool replacement)
    If arraySize <= 0
        Return
    If arraySize == 1
        If lastQueuedIndex != -1
            If replacement
                Return
        Int index = FitIndex(activeIndex, actionArray.length)
        actionArray[index] = ActionWrapper(action, true)
        lastQueuedIndex = index
    Int queueIndex = if lastQueuedIndex != -1 : lastQueuedIndex + 1, else activeIndex + 1
    queueIndex = FitIndex(queueIndex, actionArray.length)
    Int start = FitIndex(activeIndex + 1, actionArray.length)
    Int end = FitIndex(activeIndex + queueSize, actionArray.length)
    If IndexInRange(queueIndex, start, end)
        actionArray[queueIndex] = ActionWrapper(action, true)
        lastQueuedIndex = queueIndex
    Else 
        If replacement != true
            Return 
        actionArray[end] = ActionWrapper(action, true)
        lastQueuedIndex = end
    UpdateActionRefs()
```

Actions are added to the queue by overriding the data at one position with the new data. The place to add the data is at the `lastQueuedIndex`, as long as it is not -1, in which case it is added after `activeIndex`. When the `lastQueuedIndex` reaches the ending index of the “queued” section, replacement will need to be enabled to handle queue overflow as there is no more space in the “queued” section. The strict use of the sections make it so that new actions are only added to the “queued” section and do not overflow into the “previous” section.

Queue overflow, when more actions are queued than the size of the “queued” section, may be handled a few ways but will be dependent on desired behavior. The last action in the “queued” section may be overridden by any new inputs, allowing the player to change that action. New inputs may be ignored. This will depend on what should happen in your game if too many actions are queued. In the above code overflow is handled by overriding the last action in the “queued” section if the “replacement” argument is set to true.

#### Action Wrapper
When Actions are added they are placed within a wrapper struct that adds extra data in association to that Action that is used internally by the Action Queue. The wrapper holds the action and a bool “valid”, which indicates if that action in the queue has been set. This is primarily used in `Resize()`.

### Progressing The Queue
```
ProgressQueue()
    If activeIndex == lastQueuedIndex or lastQueuedIndex == -1
        Return Action()
    activeIndex = FitIndex(activeIndex + 1, actionArray.length)
    If activeIndex == lastQueuedIndex
        lastQueuedIndex = -1
    UpdateActionRefs()
    Return actionArray[activeIndex].action
```

### Get Sections
```
GetPreviousArray()
    If preserveSize < 0
        Return array<ActionWrapper>()
        array<ActionWrapper> returnArray = array<ActionWrapper>()
    If preserveSize == 1
        returnArray.add(actionArray[FitIndex(activeIndex - 1, actionArray.length)])
    Else 
        For int i = 0; i < preserveSize; ++i
            Int preserveIndex = FitIndex(activeIndex - 1 - i, actionArray.length)
            If preserveIndex == FitIndex(activeIndex+(queueSize-1), actionArray.length)
                returnArray.add(actionArray[preserveIndex])
            Else
                Break 
    Return returnArray

GetQueuedArray()
    If queuedSize < 0 
        Return array<ActionWrapper>()
    array<ActionWrapper> returnArray = array<ActionWrapper>()
    If queuedSize == 1
        returnArray.add(actionArray[FitIndex(activeIndex + 1, actionArray.length)])
    Else
        If lastQueuedIndex == FitIndex(activeIndex + 1, actionArray.length)
            Return returnArray
        For int i = 0; i < queuedSize; ++i
            Int queueIndex = FitIndex(activeIndex + 1 + i, actionArray.length)
            returnArray.add(actionArray[queueIndex])
            If queueIndex == lastQueuedIndex
                Break 
    Return returnArray
```

`GetQueuedActions()` and `GetPreviousActions()` \
Copy the above matching function but return an array of Actions rather than an array of Action Wrappers. Actions Wrappers should stay internal to Action Queue.

### Resize
```
Resize(int newPreserveSize, int newQueueSize)
    lastQueuedIndex = -1
    array<ActionWrapper> tempArray = array<ActionWrapper>()
    array<ActionWrapper> prevArray = GetPreviousArray()
    For int i = 0; i < prevArray.length; ++i
        If i != newPreserveSize
            If prevArray[i].valid
                tempArray[newQueueSize + 1 + i] = prevArray[i]
        Else 
            Break 
    tempArray[0] = actionArray[activeIndex]
    array<ActionWrapper> qArray = GetQueuedArray()
    For int i = 0; i < qArray.length; ++i
        If i != newQueueSize
            If qArray[i].valid
                tempArray[i + 1] = prevArray[i]
                lastQueuedIndex = i + 1
            Else
                Break
    preserveSize = newPreserveSize
    queuedSize = newQueueSize
    activeIndex = 0
    actionArray = tempArray
    UpdateActionRefs()
```

### Summary
 - Use a fixed size array as a queue
 - Implement index wrapping
 - Store the index of the active action
 - Use the active index and section sizes to determine the states of the surrounding actions
 - Utilize the queue in character class to manage the order of their actions based on first in first out
   - This not only manages incoming actions but holds previous actions

## Conclusion
By using a well defined key system along with a search tree data structure, attacks may be stored in a way that allows for different paths to be taken based on different inputs and character context, resulting in unique attack chains. 

Once the right attack is selected it can be passed to a custom queue that will help in outputting the attacks at the right time and correct order. The queue can also provide data from past actions to further help provide context when choosing the next attack. 

![ActionManagement drawio](https://github.com/user-attachments/assets/a16a314a-b1d8-4d7f-82a7-d04a58ba5648)

Ultimately this setup will allow for a customizable system that will allow a wide variety of combat systems, from simple three attack combos to input heavy button masher combos.
