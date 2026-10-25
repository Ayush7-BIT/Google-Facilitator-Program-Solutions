## Task 1: Create the core app

Paste this into the main prompt box and run it:

```text
Build a single-page web app called "Fantasy Character Creator". It has one button labeled "Generate Character". When clicked, it uses the Gemini API to create a random fantasy character with a unique Name and a Class (for example Mage, Rogue, Warrior, Paladin, Ranger) and displays them clearly on the page, labeled "Name" and "Class". Every click must produce a new, different character. Show a loading state while generating and handle errors gracefully.
```

## Task 2


```text
Add a character portrait feature. Below the character details, add a "Portrait" section with two buttons labeled exactly "Generate" and "Regenerate". "Generate" creates a cartoon / video game style portrait of the currently displayed character (use their Name and Class in the image prompt) using the Gemini API's image generation, and shows it. "Regenerate" creates a new, different portrait for the same character. Show a loading state. When a new character is generated, clear the old portrait. Keep all existing features working.
```


## task 3

```text
Restyle the whole app like an "ancient alchemist's workbench": a dark, textured background (aged dark wood or stone texture), parchment, brass and amber accents, and a stylized fantasy-style font (for example Cinzel Decorative or MedievalSharp from Google Fonts) for the character's name. Keep all existing features working.
```

## task 4

```text
Add a button labeled "Generate Backstory". When clicked, it uses the Gemini API to write a unique, one-to-two-sentence origin story for the character currently on screen, based on their Name and Class, and displays it below the character. Clear the backstory when a new character is generated. Keep everything else working.
```

## task 5

```text
Reformat the entire UI into a single bordered player card, like an RPG trading card. The card contains the portrait, the character's name in the fantasy font, the class, randomly generated stats for Health, Mana and Strength (new random numbers from 1 to 100 for each new character), and the backstory. Add a button labeled "Save to Deck" that saves the current character (name, class, stats, portrait and backstory) into a list called "My Deck", displayed on the page below the card. Each saved character appears as a small card in "My Deck". Keep all previous features and the alchemist theme working.
```


