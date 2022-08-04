# TSP Quiz Share format - Version 2
This specification exist to provide open access to the data format which TSP Quiz
uses to share quiz between users. With this information other apps/websites that
work with signs from Teckenspårkslexikon can implement export/share to TSP Quiz.

Overall the format specifies the exact order of answer options and which one is
the correct one.

When extensions are added, the version number will be incremented.

* [ChangeLog](CHANGES.md)
* [Version 1](SHARE_FORMAT2.md)

## 1. Data Format
Build a data structure based on ShareData as root element.

To obtain word ids, go to [https://teckensprakslexikon.su.se/](https://teckensprakslexikon.su.se/)
and search for signs there. On the page of a sign you see a field called "ID-nummer" which
contains the word id. This ID is also included in the page URL.

```ts
// All number types should contain integers

enum QuestionType {
    GuessFromVideo = 0, // User sees video of correct_index and select which Swedish word to answer.
    GuessVideoFromWord = 1, // User sees the Swedish word of correct_index and can see all videos.
    TypeFromVideo = 2, // User sees video of correct_index and should type the Swedish word.
    SignFromWord = 3, // User sees the Swedish word of correct_index and should sign it for themself and then get to see the video.
}
interface ShareQuestion {
    type : QuestionType; // Type of question
    words : String[]; // Array of word ids. Eg. '05382' for the word Teckenspråk.
    correct_index : number; // 0-based index to which of the words that is correct
}
interface ShareOptions {
    name : string; // Empty string or name of quiz, max 50 characters.
    timestamp : number; // Unix timestamp in seconds of when the quiz was created.
    altWords : boolean; // True to resolve and display alternative translations for words in this quiz, false to disable. (recommended)
    altIncludeUncommon : boolean; // True to include uncommon words when resolving alternative translations, false to disable. (not recommended)
}
interface ShareData {
    version : number; // Share data format version. Should be 1 or 2. (note: does not corresponds to TSP Quiz versions)
    options : ShareOptions|null; // Should be null if version is 1 and non-null if version is 2.
    questions : ShareQuestion[]; // Array of questions in the quiz
}

// For QuestionType TypeFromVideo and SignFromWord, there should be exactly
// one element in ShareQuestion.words and correct_index should be 0.
//
// For QuestionType QuessFromVideo and GuessVideoFromWord, TSP Quiz have
// 4 words. But at least one is required by the format and you could be
// creative and use 3 or 5 words if you wish, but the UI of TSP Quiz
// has been optimized for no more than 4 words.

```

### What is a word?
In lexikon and TSP Quiz a word contains both a Swedish meaning (name) and a
video along with meta information about which category it belongs to etc.
Some words have alternative Swedish meanings coded onto this same word ID.

TSP Quiz may in some cases show alternative videos for a word. TSP Quiz does
this by searching for words in lexikon that have the same Swedish meaning as
the word given by the question words[] array.


## 2. Base64 encode
Encode the root element to Json string and then base64 encode that string.

```ts
let data : ShareData = { version: 1, questions: [..] };
let data_str = JSON.stringify(data);
let base64_str = btoa(data_str);
```

## 3. Share link
Put the base64 encoded string on the link

```ts
let url = new URL('https://tspquiz.se/app');
url.searchParams.append('loadQuiz', base64_str);
url.hash = '/start';
let share_link = url.toString();

// Now share share_link to your friends etc.
```

## Example
This example is in JavaScript.

```js
// Build quiz data
let data = {
    version: 1,
    questions: [
        {
            type: 0, // guess from video
            words: ['05382', '05196', '08156', '04568'],
            correct_index: 2,
        },
    ],
};
// Base64 encode
let data_str = JSON.stringify(data);
let base64_str = btoa(data_str);
// Build url
let url = new URL('https://tspquiz.se/app');
url.searchParams.append('loadQuiz', base64_str);
url.hash = '/start';
let share_link = url.toString();
console.log(share_link);

// =>
// https://tspquiz.se/app?loadQuiz=eyJ2ZXJzaW9uIjoxLCJxdWVzdGlvbnMiOlt7InR5cGUiOjAsIndvcmRzIjpbIjA1MzgyIiwiMDUxOTYiLCIwODE1NiIsIjA0NTY4Il0sImNvcnJlY3RfaW5kZXgiOjJ9XX0%3D#/start
```

## Validation
TSP Quiz contains a format validator that validates the data format before starting a quiz.
If the format is wrong, the app will complain and not start the quiz.

If the quiz contains non-existent word IDs, that will not be checked by the validator and the
app will fail to load that answer option as words are lazy loaded.

So to validate a quiz you need to step through all questions to ensure it loads properly.
