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
