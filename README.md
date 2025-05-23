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
Use an array. Store the index of the active attack and increment the index as attacks are performed. Reset the index to 0 when the end is reached, or set to 0 after a short delay to reset the chain after a lack of input so that the chain does not start in the middle after an extended period of no input/non-attack inputs.

To increase attack variety multiple arrays may be used and swapped as the character equips different weapons, changes states (such as being in the air), or if the player provides a different input for a different type of attack.

![LinearCombo](https://github.com/user-attachments/assets/ca28b700-172d-4828-9caa-ca636e81e801)
