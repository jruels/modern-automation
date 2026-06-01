# Python Guessing Game — Beginner's Guide

In this lab you will build a number-guessing game from scratch using Python. By the end you will have a working script that picks a random number, accepts guesses from the player, and tells them whether to guess higher or lower.

---

## What you will learn

- How to generate a random number with the `random` module
- How to collect input from the user
- How to compare numbers with `if / elif / else`
- How to repeat code with a `for` loop

---

## Prerequisites

- Python 3 installed
- VS Code with the Python extension installed

---

## Step 1 — Create the file

Create a new file called `guessing_game.py` in any folder you like.

```bash
touch guessing_game.py
```

Open it in your editor and leave it blank for now.

---

## Step 2 — Import the `random` module

Python's built-in `random` module lets you generate random numbers. Add this as the very first line of your file:

```python
import random
```

---

## Step 3 — Generate a random number

Use `random.randint(low, high)` to pick a random whole number between 1 and 10 (inclusive).

```python
import random

secret_number = random.randint(1, 10)
```

`secret_number` now holds a different value every time the script runs.

---

## Step 4 — Add a loop for five attempts

The player gets exactly five guesses. A `for` loop is perfect here — it runs a fixed number of times.

```python
import random

secret_number = random.randint(1, 10)

for attempt in range(1, 6):   # 1, 2, 3, 4, 5
    print(f"Attempt {attempt} of 5")
```

Run the script now to confirm the loop counts up correctly by clicking the **Play** button (▶) in the top-right corner of VS Code.

Expected output:
```
Attempt 1 of 5
Attempt 2 of 5
Attempt 3 of 5
Attempt 4 of 5
Attempt 5 of 5
```

---

## Step 5 — Ask the player for a guess

Inside the loop, use `input()` to collect the player's guess. `input()` always returns a string, so wrap it with `int()` to convert it to a number.

```python
import random

secret_number = random.randint(1, 10)

for attempt in range(1, 6):
    print(f"Attempt {attempt} of 5")
    guess = int(input("Guess a number between 1 and 10: "))
```

---

## Step 6 — Compare the guess to the secret number

Add an `if / elif / else` block to give the player feedback after each guess.

```python
import random

secret_number = random.randint(1, 10)

for attempt in range(1, 6):
    print(f"Attempt {attempt} of 5")
    guess = int(input("Guess a number between 1 and 10: "))

    if guess == secret_number:
        print("You win!")
    elif guess < secret_number:
        print("You're thinking too small.")
    else:
        print("You're thinking too large.")
```

Try running the script and making a few guesses. You will see feedback after every attempt.

---

## Step 7 — Stop the loop when the player wins

Right now the game keeps asking for guesses even after a correct answer. Add a `break` statement to exit the loop immediately when the player wins.

```python
import random

secret_number = random.randint(1, 10)

for attempt in range(1, 6):
    print(f"Attempt {attempt} of 5")
    guess = int(input("Guess a number between 1 and 10: "))

    if guess == secret_number:
        print("You win!")
        break                  # stop the loop — no more guesses needed
    elif guess < secret_number:
        print("You're thinking too small.")
    else:
        print("You're thinking too large.")
```

---

## Complete script

Here is the full, finished script:

```python
import random

secret_number = random.randint(1, 10)

for attempt in range(1, 6):
    print(f"Attempt {attempt} of 5")
    guess = int(input("Guess a number between 1 and 10: "))

    if guess == secret_number:
        print("You win!")
        break
    elif guess < secret_number:
        print("You're thinking too small.")
    else:
        print("You're thinking too large.")
```

Run it by clicking the **Play** button (▶) in the top-right corner of VS Code.

Sample session:
```
Attempt 1 of 5
Guess a number between 1 and 10: 5
You're thinking too small.
Attempt 2 of 5
Guess a number between 1 and 10: 8
You're thinking too large.
Attempt 3 of 5
Guess a number between 1 and 10: 6
You win!
```

---

## Bonus challenge

Update the game so that if the player uses all five attempts without winning, it reveals the secret number. Add this code after the `for` loop:

```python
import random

secret_number = random.randint(1, 10)
won = False                           # track whether the player guessed correctly

for attempt in range(1, 6):
    print(f"Attempt {attempt} of 5")
    guess = int(input("Guess a number between 1 and 10: "))

    if guess == secret_number:
        print("You win!")
        won = True
        break
    elif guess < secret_number:
        print("You're thinking too small.")
    else:
        print("You're thinking too large.")

if not won:
    print(f"Out of attempts! The number was {secret_number}.")
```

**How it works:**

1. `won = False` — a flag variable set before the loop starts.
2. When the player guesses correctly, `won` is set to `True` before `break`.
3. After the loop, `if not won` is only `True` when all five attempts were used without a correct guess.

---

## Congrats!

You built a fully functional guessing game using:

- `import random` and `random.randint()`
- `input()` and `int()` for user input
- `if / elif / else` for decision making
- `for` loops and `break` for controlled repetition
