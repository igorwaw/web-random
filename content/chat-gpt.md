Title: How to use ChatGPT and Github Copilot to code effectively
Date: 2023-04-12 17:30
Status: draft
Category: random
Tags: ai
Slug: chat-gpt

If you're a software developer, you might be worried that generative AI will take your job. Well, worry not. Yes, ChatGPT can generate a (sometimes) working code in many programming languages, but that's the easiest part of programming. Start to worry when it learns to debug.

OK, now seriously: I believe that generative AI is a really useful tool for developers, not a replacement for us. Just like we all became much more effective with Google and modern IDEs compared to reference books and editors from the past, we can now code faster when assisted by  the AI. Writing the right prompts and combining AI-generated code with your own knowledge are the keys to success. ChatGPT could generate the code from start to finish and so could a human developer, but the fastest way is to use the strong points of each: let the human do the thinking and the computer extend the human's memory. 

And before we start: yes, I know that's not such thing as AI, it's just a large language model. But the name stuck, so I'll use it.


## 1. Start with generating boilerplate code

How do I connect to MongoDB? How do I parse JSON? How to read config from environment variables and command line? A large part of every application is a standard code we call boilerplate. It's an often repeated joke that junior developer's job is mostly copy-pasting code from StackOverflow. The hidden truth  is: experienced developers do the same. Or they copy-paste from their own code written in the past. And there's nothing wrong about it, computers are good at remembering, we don't need to clog our tiny human memory.

Generative AI just makes the process slightly faster because you can use a conversation-like interface. The code can be personalized, eg. it would connect to your specific webservice, not some example webservice. And you can ask it to explain parameters.

The easiest way is to use ChatGPT to generate the Lego bricks and then assemble them on your own. Let's say your task is to read some numbers from Postgres and create plots. You might do it like this:

- ask ChatGPT how to connect to Postgres in Python,
- ask how to write a SQL query that gathers your data,
- ask how to create a plot with Matplotlib,

then combine those blocks yourself. Note that I assumed you know what tools you need. Specific questions work better, so try to provide all the information you have. But if you're exploring a new field, don't be afraid to ask generic questions. They will eventually get you to the result too, just with more time.

Github Copilot works in a similar way, but if you use an IDE plugin, it's more convenient. You can write a comment saying what you want to do, eg. "//check if file exists", or even faster, start writing the code yourself and in a second you'll see a suggestion you can accept with Tab or ignore. And just like ChatGPT, it's much better with standard tasks than with anything even slightly complex.


## 2. Dealing with errors in the generated code

There is one important caveat when using AI instead of a search engine. A top-voted reply from StackOverflow is almost guaranteed to be a proper, working code. ChatGPT will sometimes give you a code that doesn't compile or even doesn't answer your question at all. And it will do so with an utmost confidence. But you can ask it to correct the errors. It will apologize and rewrite the part. It helps if you understand the code and can type "you made a mistake because this variable  is used before initialization", but even if you don't and write "I tried running your code and got this error: syntax error at line 11", ChatGPT will try to correct. Splitting your problem into smaller tasks also helps.

You can also find it's faster to fix the code yourself or even disable Copilot if the suggestions are completely wrong and only distracting. 


## 3. Iteration

I'm sure you know what's iteration. But here, I don't mean writing a loop, I mean an iterative process. Sticking to the same example as before: ChatGPT is completely capable of writing a simple application like this  (but not the bigger, more realistic application though), you don't have to write a single line of code.  But don't even try describing the whole application in one go, even for this trivial example it's impossible. What you could do instead is start simple and build on top of the previous steps:

- get a working DB connection first,
- then ask GPT to get the specific data using this connection, 
- then to use the data to create a simple plot,
- then to add plot elements such as legend, grid etc.

Remember to test every step.

## 4. Naming and versioning

You can name the code blocks in your GPT session. Got a working DB connection? Write: "OK, let's name that code DB01". You can later ask the AI to do something with a variable "row" from "DB01". Or, if the experiments reach a dead end (and they will), revert to DB01. Using names and version number is much easier and less error-prone than writing "please use the version we previously talked about".


## 5. Dealing with outdated information

ChatGPT's language model was trained in 2021. What if you want to use a library that appeared later or a newer version of a Web API? Simply paste the relevant part of the docs into your conversation and ask GPT to include the new information when rewriting the code. Note there is a limit on how much new information it can handle (about 4000 words). All the more reason to split your complicated problem into smaller tasks. Plus, the new information is only available in one specific conversation. You can however go back to your old conversations. 


## 6. Refactoring

AI isn't limited to writing new code. You can use it to improve existing functions and classes, just paste the code into ChatGPT prompt.  The task is even easier with Copilot since it's controlled from your IDE. You can even try asking the AI "make this function better" - sometimes the simplest question works! Most likely, however, you will get better results if you know what to do. Hint: since the language model will often create wrong code, start your experiments by asking it to generate assertions and unit tests. 

## 7. Careful with secrets

OpenAI, the company behind ChatGPT, uses conversations to train new language models. The employees can read everything you typed or pasted. To be fair, they probably won't, given the huge amount of data ChatGPT receives every day, the probability of reading your specific conversation is low. Even lower that they read all of your input, so that's yet another reason to split your problem. But it might be against your company's policy to paste even a small amount of confidential code to external systems. Be especially careful not to enter passwords and API keys. If you accidentaly leaked something, you cannot remove only a specific part of your conversation - you can only request to delete your account with all the associated data.

Github Copilot was trained on opensource projects hosted on Github. Even if you're not using it, it might already ingested your code (you can opt out). If you do use it, you see the suggestions in your own IDE, but the processing is done in the cloud, meaning the code you type is sent to external systems. You're leaking it all the time! Again, this could be against your company's policy.


## 8. Coding in the language you don't know

ChatGPT can generate code in several programming languages: Java, Python, C++, JavaScript, Ruby, PHP, Swift  plus SQL queries and HTML with CSS. It also know thousands of libraries and APIs and you can include your own. Say you're a Python programmer, but just once you need to write some simple JavaScript or modify a bit of Ruby? You can do it quckly with an AI-assisted process. You know what's a loop and conditional, some instructions will look familiar - that means you can understand the code and formulate your questions in the right way. A non-coder wouldn't be able to do the same, but it greatly extends your powers if you're already a programmer. Does it mean that every developer can now become a universal developer? Not quite, the further you stray from your field, the less you'll be able to do. With some luck, a frontend developer can generate some embedded C++ code, but won't necessarily know how to test and debug it. 


## 9. ChatGPT can help with auxilary tasks too

Do you need to run your app in a Docker container or Kubernetes pod? Do you want to configure continous integration or a good old build system? Need a reverse proxy for your web service? Perhaps you never used  git (where have you been for the last 15 years?) but heard it's something you should use for your code? ChatGPT can generate commands and config files for you. The usual warning applies: there's no guarantee they will be correct, but often they are.

Another use is generating example data. UI designers always keep their "lorem ipsum" placeholder text ready to paste, but what if you need a SQL table with a specific structure, CSV with a list of fake addresses or something more unusual like a sensor data?  Sure, you can google for it, but the AI can make one customized for you.


## 10. Still not sure how ChatGPT can help? Ask it

Just for fun, I asked "how to use ChatGPT to code more effectively?" and the result was quite close to the list I wrote.


