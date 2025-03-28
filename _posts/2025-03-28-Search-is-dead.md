# Search is Dead, Long Live Search: How I Built a Better Job Matching Tool

**TL;DR**: I am tapping the Hacker News Firebase API and RemoteOK.com to grab job postings, Google Gemini for the text embeddings with cosine similarity distance from Lesson 9, and Google gemini-2.0-flash for the skills development plan generation. All running on Google's free tier while they "improve their products" (you are welcome Google!).

## The Death of Traditional Search

Let's be honest - traditional keyword search for jobs can be challenging. You write "ML" instead of "Machine Learning"? Too bad, no match. You describe "building scalable systems" but don't explicitly say "system architecture"? The algorithm doesn't care about your actual skills.

Meanwhile, the big job sites claim to use AI matching but give you zero insight into why you matched with certain jobs. It's a black box that leaves you wondering if you should rewrite your entire resume just to game their system.

I decided we could do better.

## The Resurrection: Hybrid Search

I started by playing with vector embeddings and cosine similarity (shout out to Lesson 9) and realized there was a sweet spot between old-school keyword matching and pure AI magic.

Here's the secret sauce:
1. Use Google Gemini's embeddings to understand the semantic meaning of resumes and job postings
2. Add explicit skill extraction to identify specific technologies and requirements
3. Mix in preference matching for things like remote work and location
4. Combine these signals in a weighted formula (70% semantic, 30% explicit)
5. Generate actual human-readable explanations of why you matched

## How I Built It

The tech stack is straightforward but powerful:

- **Data Sources**: 
  - Hacker News Firebase API for "Who is Hiring" job postings
  - RemoteOK.com for remote-friendly positions (new addition; thank you @shuane!)
- **Embeddings**: Google Gemini API for converting text to vector embeddings
- **Matching**: Cosine similarity to compare resume and job vectors
- **Skills Analysis**: Custom regex-based skill extraction (with a ton of optimization to avoid timeouts)
- **Recommendations**: Upgraded to gemini-2.0-flash for even better personalized skill development plans
- **Frontend**: Streamlit because I wanted to ship this fast
- **Cost**: Running on Google's free tier while they "improve their products" (read: train on our data)

The code is modular with proper separation of concerns - embedding service, skill extraction, matching algorithm, and explanation generation all nicely decoupled.

## Show Me The Money: Real Results

When you paste your resume and hit the button, you get results like this:

```
Match #1: Fractional Engineer at Portal Stargaze - 77.2% match

This position at Portal Stargaze is a 77.2% overall match.
Job title: Fractional Engineer
Skills match: 67.5%
Preference match: 100.0%
Remote work: Yes
Location: Fractional Engineer ✓

Your skills that match: machine learning, python, r, react, aws

Skills to develop: go, airflow, etl, rest, data science, ai

AI GENERATED: Okay, here's a comprehensive learning plan designed for a Fractional Engineer role with a focus on Go, Airflow, ETL, REST APIs, and Data Science. This plan is structured to be flexible and adaptable based on your existing knowledge and the specific requirements of the fractional position.

Overall Goal: To develop a strong foundation and practical skills in Go, Airflow, ETL processes, REST API development, and introductory data science techniques, enabling you to contribute effectively to a fractional engineering role.

Target Audience: This plan assumes you have some programming experience (ideally with Python or a similar language) and a general understanding of software development concepts. It's designed to be adaptable for varying levels of prior knowledge.

Plan Structure:

The plan is broken down into modules for each skill, with estimated time commitments. It's important to prioritize based on the job requirements and your existing skills. Consider starting with Go and then moving to the other modules in a logical order based on dependencies.

I. Go (Programming Language)

Goal: Become proficient in Go syntax, data structures, control flow, concurrency, and standard library.

Estimated Time: 2-4 weeks (depending on prior programming experience)

Week 1: Fundamentals

Topics:
Introduction to Go (history, features, installation, go env, go version)
Basic Syntax: Variables, data types (integers, floats, strings, booleans), operators
Control Flow: if, else, for, switch, defer
Functions: Defining functions, parameters, return values, multiple return values
Packages: Creating and using packages, import statement
Pointers
Resources:
"A Tour of Go" (official interactive tutorial): https://go.dev/tour/welcome/1
"Effective Go": https://go.dev/doc/effective_go
Go by Example: https://gobyexample.com/
Practice:
Solve basic coding problems on platforms like HackerRank, LeetCode, or Codewars using Go.
Write simple Go programs to manipulate data, perform calculations, and interact with the command line.
Week 2: Data Structures and Concurrency

Topics:
Arrays, Slices, Maps
Structs and Methods
Interfaces
Error Handling
Concurrency: Goroutines, Channels, Mutexes, WaitGroups
Resources:
Go documentation on data structures: https://go.dev/
"Concurrency in Go" by Katherine Cox-Buday (book, optional)
Effective Go (Concurrency section)
Practice:
Implement data structures like stacks, queues, and linked lists in Go.
Write concurrent programs that perform tasks in parallel using goroutines and channels.
Handle errors gracefully in your programs.
Week 3-4: Advanced Topics and Project
Topics:
Testing (unit tests, integration tests)
Standard Library: net/http, encoding/json, io, os, time
Dependency Management: Go Modules (go mod)
Reflection (optional, but useful for some advanced scenarios)
Resources:
Go documentation on the standard library.
"Go Programming Blueprints" by Mat Ryer (book, optional, focuses on project-based learning)
Project:
Build a simple REST API: Create a basic API that performs CRUD operations (Create, Read, Update, Delete) on a data source (e.g., in-memory, file-based, or a simple database).
Build a command-line tool: Develop a CLI tool that automates a task (e.g., data transformation, file processing, or interacting with an API).
```

Look at that! The system found matches for machine learning, Python, R, React, and AWS skills, but identified gaps in Go, Airflow, ETL, REST APIs, and data science. Then gemini-2.0-flash generated a detailed, week-by-week learning plan to help you develop those missing skills.

This isn't just "you should learn Go" - it's a complete curriculum with resources, practice exercises, and a logical progression. That's the difference between a generic job board and a tool that actually helps you grow your career.

## The Biggest Challenges

Building this wasn't all smooth sailing:

1. **Regex Hell**: My first skill extraction patterns caused catastrophic backtracking on large job descriptions and while that failure is no longer a concern, it is still in need of improvement
2. **API Costs**: Using caching to keep any potential long-term Gemini API costs reasonable, although the free tier helps greatly for now!
3. **Unstructured Data**: Parsing Hacker News and RemoteOK job posts requires different approaches - the formats are wildly different
4. **Balancing Signals**: More testing is needed to find the "best" weights between semantic matching and explicit skills

## Try It Out For Yourself

The tool is live on Streamlit. Just paste or upload a plain text version of your resume, set your preferences (remote work, locations), and hit the button. It now uses gemini-2.0-flash to generate even better personalized skill development plans for any missing skills.

You can also choose your job source - Hacker News for the classic "Who is Hiring" posts or RemoteOK for remote-friendly positions. More sources coming soon!

The UI is straightforward - resume and preferences on the left, results on the right (or top/bottom on mobile).

## What's Next?

I'm working on:
- Adding even more job sources beyond Hacker News and RemoteOK
- Building a skill knowledge graph to better understand relationships between technologies
- Implementing career path recommendations based on your current skills
- Creating more detailed learning roadmaps
- Generating content to assist with interview preparations 
- Maybe a subscription model once Google starts charging me for all these API calls

## The Future of Search is Hybrid

Pure keyword search is dead. Pure AI "black box" matching isn't good enough either. The future belongs to systems that combine the best of both worlds - the semantic understanding of AI with the precision and explainability of traditional methods.

This is just a side project for now, but I think it points to where all search is heading - not just for jobs, but for any domain where both meaning and specifics matter.

Let me know what you think - I'm considering turning this into a proper product if there's enough interest. And hey, get it while Google's still offering that free tier!
