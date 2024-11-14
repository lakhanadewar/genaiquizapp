# Interactive Quiz Bot using Telegram and Python

## Overview

This document provides a comprehensive guide to set up and use the interactive quiz bot that generates quizzes from PDF files. The bot uses the Telegram Bot API, PyMuPDF for PDF processing, and Python for text processing and quiz generation.

## Prerequisites

- Python 3.7+
- Telegram account to create a bot
- Required Python libraries:
  - `python-telegram-bot`
  - `PyMuPDF`

## Setup Instructions

### Step 1: Install Required Libraries

Install the required Python libraries using pip:

```bash
pip install python-telegram-bot PyMuPDF
```

### Step 2: Create a Telegram Bot

1. Open Telegram and search for the BotFather.
2. Start a chat with BotFather and use the `/newbot` command to create a new bot.
3. Follow the instructions to set the bot's name and username.
4. After creation, you will receive a token. Save this token for later use.

### Step 3: Create the Python Script

Create a Python script (e.g., `quiz_bot.py`) and include the following code:

```python
import os
import fitz
import random
from telegram import Update, Document, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes, CallbackQueryHandler

TOKEN = 'YOUR_BOT_TOKEN'

class QuizState:
    def __init__(self):
        self.current_question = 0
        self.score = 0
        self.questions = []

def split_into_sentences(text: str) -> list:
    """Split text into sentences without using NLTK"""
    endings = ['. ', '! ', '? ', '.\n', '!\n', '?\n']
    sentences = []
    current = ""
    
    for char in text:
        current += char
        if any(current.strip().endswith(end.strip()) for end in endings):
            if len(current.strip()) > 20:
                sentences.append(current.strip())
            current = ""
    
    if len(current.strip()) > 20:
        sentences.append(current.strip())
    
    return sentences

def generate_quiz_questions(text: str, num_questions: int) -> list:
    try:
        sentences = split_into_sentences(text)
        
        if not sentences:
            raise ValueError("No valid sentences found in the text")

        questions = []
        used_sentences = set()
        
        for i in range(min(num_questions, len(sentences))):
            available_sentences = [s for s in sentences if s not in used_sentences and len(s.split()) > 5]
            if not available_sentences:
                break
                
            sentence = random.choice(available_sentences)
            used_sentences.add(sentence)
            
            words = [word.strip('.,!?()[]{}":;') for word in sentence.split()]
            key_words = [w for w in words if len(w) > 4 and w.isalnum()]
            
            if not key_words:
                continue
                
            key_word = random.choice(key_words)
            question_text = sentence.replace(key_word, "_____")
            
            wrong_options = []
            possible_wrong_words = [w for w in key_words if w != key_word]
            
            if possible_wrong_words:
                wrong_options.extend(random.sample(possible_wrong_words, min(3, len(possible_wrong_words))))
            
            while len(wrong_options) < 3:
                variation = key_word + random.choice(['s', 'ed', 'ing', 'er'])
                if variation not in wrong_options:
                    wrong_options.append(variation)
            
            options = wrong_options + [key_word]
            random.shuffle(options)
            
            correct_answer = options.index(key_word)
            
            questions.append({
                'question': question_text,
                'options': options,
                'correct': correct_answer
            })
        
        return questions

    except Exception as e:
        print(f"Error in generate_quiz_questions: {e}")
        return [{
            'question': "What is this text about?",
            'options': ["Main Topic", "Secondary Topic", "Related Topic", "Other Topic"],
            'correct': 0
        }]

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text('Hello! Send me a PDF file, and I will generate an interactive quiz for you.')

async def handle_document(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    try:
        document: Document = update.message.document
        if document.mime_type == 'application/pdf':
            await update.message.reply_text('Processing your PDF file... Please wait.')
            
            file = await document.get_file()
            file_path = f"{document.file_id}.pdf"
            await file.download_to_drive(file_path)

            text = extract_text_from_pdf(file_path)
            
            if len(text.strip()) == 0:
                await update.message.reply_text('The PDF appears to be empty or unreadable.')
                return

            context.user_data['extracted_text'] = text
            context.user_data['file_path'] = file_path
            await update.message.reply_text('How many questions would you like to generate? (1-5 recommended)')
        else:
            await update.message.reply_text('Please send a valid PDF file.')
    except Exception as e:
        await update.message.reply_text(f'An error occurred while processing the file: {str(e)}')

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if 'extracted_text' in context.user_data:
        try:
            num_questions = int(update.message.text)
            if num_questions <= 0:
                await update.message.reply_text('Please enter a positive number.')
                return
            
            if num_questions > 10:
                await update.message.reply_text('Maximum 10 questions allowed. Generating 10 questions...')
                num_questions = 10
            
            text = context.user_data['extracted_text']
            questions = generate_quiz_questions(text, num_questions)
            
            if not questions:
                await update.message.reply_text('Could not generate questions from the text. Please try with different content.')
                return
                
            context.user_data['quiz_state'] = QuizState()
            context.user_data['quiz_state'].questions = questions
            
            await send_question(update, context)
            
            file_path = context.user_data['file_path']
            if os.path.exists(file_path):
                os.remove(file_path)
                
        except ValueError:
            await update.message.reply_text('Please send a valid number between 1 and 10.')
    else:
        await update.message.reply_text('Please send a PDF file first.')

async def send_question(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    quiz_state = context.user_data['quiz_state']
    current_q = quiz_state.questions[quiz_state.current_question]
    
    keyboard = []
    for idx, option in enumerate(current_q['options']):
        keyboard.append([InlineKeyboardButton(f"{chr(65+idx)}) {option}", 
                                            callback_data=str(idx))])
    
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    question_text = (f"Question {quiz_state.current_question + 1}:\n\n"
                    f"{current_q['question']}\n\n"
                    f"Your score: {quiz_state.score}/{quiz_state.current_question}")
    
    if update.message:
        await update.message.reply_text(question_text, reply_markup=reply_markup)
    else:
        await update.callback_query.message.edit_text(question_text, reply_markup=reply_markup)

async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()
    
    quiz_state = context.user_data['quiz_state']
    current_q = quiz_state.questions[quiz_state.current_question]
    
    if int(query.data) == current_q['correct']:
        quiz_state.score += 1
        feedback = "✅ Correct!"
    else:
        feedback = f"❌ Wrong! The correct answer was: {current_q['options'][current_q['correct']]}"
    
    await query.message.reply_text(feedback)
    
    quiz_state.current_question += 1
    if quiz_state.current_question < len(quiz_state.questions):
        await send_question(update, context)
    else:
        final_score = f"Quiz completed!\nFinal score: {quiz_state.score}/{len(quiz_state.questions)}"
        await query.message.reply_text(final_score)
        context.user_data.clear()

def extract_text_from_pdf(file_path: str) -> str:
    try:
        doc = fitz.open(file_path)
        text = ""
        for page in doc:
            text += page.get_text()
        doc.close()
        return text
    except Exception as e:
        print(f"Error extracting text from PDF: {e}")
        return ""

def main():
    try:
        application = ApplicationBuilder().token(TOKEN).build()

        application.add_handler(CommandHandler("start", start))
        application.add_handler(MessageHandler(filters.Document.PDF, handle_document))
        application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
        application.add_handler(CallbackQueryHandler(button_callback))

        print("Bot is running...")
        application.run_polling()
    except Exception as e:
        print(f"Error starting the bot: {e}")

if __name__ == '__main__':
    main()
```

### Explanation of the Code

1. **Imports and Initialization**:
   - Import necessary libraries.
   - Set up the bot token.

2. **QuizState Class**:
   - Maintains the state of the quiz, including the current question, score, and list of questions.

3. **Text Processing Functions**:
   - `split_into_sentences`: Splits text into sentences without using NLTK.
   - `generate_quiz_questions`: Generates quiz questions from the text.

4. **Telegram Bot Handlers**:
   - `start`: Sends a welcome message.
   - `handle_document`: Processes the uploaded PDF and extracts text.
   - `handle_message`: Handles user input for the number of questions.
   - `send_question`: Sends a quiz question to the user.
   - `button_callback`: Handles button clicks for quiz answers.

5. **PDF Text Extraction**:
   - `extract_text_from_pdf`: Extracts text from the uploaded PDF file.

6. **Main Function**:
   - Sets up the Telegram bot and handlers.
   - Starts the bot.

### Running the Bot

1. Replace `'YOUR_BOT_TOKEN'` with your actual bot token in the script.
2. Run the script:
```bash
python quiz_bot.py
```
3. Interact with your bot on Telegram:
   - Start the bot with the `/start` command.
   - Upload a PDF file.
   - Enter the number of questions you want to generate (1-10).
   - Answer the quiz questions interactively.


