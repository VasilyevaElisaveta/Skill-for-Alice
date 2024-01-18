def format_text(text: str) -> str:
    """
    Format the text string by removing all punctuation characters,
    spaces, line breaks and other special characters.
    Param: text (string): The source text to be formatted.
    Return: formatted_text (string): Formatted text with all punctuation characters, 
    spaces, line break characters, and other special characters are replaced with an empty string.
    """
    formatted_text = text.strip().lower()
    for symbol in punctuation + whitespace + "…" + "—":
        formatted_text = formatted_text.replace(symbol, "")
    return formatted_text


    def get_poem(poem_title):
    """
    Gets the text of the poem from Yandex.Cloud storage.
    Param: poem_title (string): The title of the poem.
    Return: poem_text (string or None): The text of the poem if the poem is found. 
    If no poem is found, the value None is returned.
    """
    # URL of the poem file in Yandex.Cloud
    url = f"https://storage.yandexcloud.net/{CLOUD_ID}/{poem_title}.txt"
    # Headings for the enquiry
    headers = {"Authorization": f"Bearer {YANDEX_STORAGE_TOKEN}"}
    # Send a GET request to Yandex.Cloud
    response = requests.get(url, headers=headers)
    # If successful, return the text of the poem
    if response.status_code == 200:
        response.encoding = "utf-8"
        return response.text.split("\n")
    else:
        return None


        

    def welcome_message(event):
    """
    Returns a welcome message.

    Param:event (dict): The event object.
    Returns:response (dict): The response object containing the welcome message and buttons.
    """
        return {
        "version": "1.0",
        "session": event["session"],
        "response": {
            "text": "Привет, я помогаю учить стихи\nНазови стих, который тебе надо выучить.",
            "buttons": [
                {
                    "title": "Помощь"
                }
            ],
            "end_session": False,
        },
    }





    def handle_learning(session, event):
    """
    Handles the learning phase that the user is in.

    :param session: session attributes
    :param event: event
    :return: response for the user
    """
    # Checking if the training has started
    if session.get("learning_started"):
        session_is_end = False
        learning_output = session.get("learning_output")
        poem_text = session["poem_text"]
        user_input = format_text(event["request"]["original_utterance"])

        # If the learning phase is the full text
        if learning_output == "full_text":
            correct_text = format_text("".join(poem_text))
            if user_input != correct_text:
                response = "Пожалуйста, повторите стихотворение правильно."
            else:
                response = "Обучение окончено!"
                session_is_end = True

        # If the learning phase is the repetition of a string
        elif learning_output == "repeat_line":
            correct_text = format_text(
                "".join(poem_text[session["current_part"]: session["current_part"] + 2])
            )

            # If the user's input matches the correct text
            if user_input == correct_text:
                session["current_part"] += 2
                if session["current_part"] >= len(poem_text):
                    session["learning_output"] = "full_text"
                    response = "Отлично! А теперь повтори стихотворение полностью."
                elif session["current_part"] >= 4:
                    session["learning_output"] = "repeat_block"
                    response = "\n".join(poem_text[:session["current_part"]])
                # If the current part is greater than or equal to the length of the verse text
                else:
                    response = "\n".join(
                        poem_text[session["current_part"]: session["current_part"] + 2]
                    )
            elif user_input.lower() in ('повтори', 'повтористрочки', 'повтористроку', 'можешьповторить'):
                if session["current_part"] >= 4:
                    session["learning_output"] = "repeat_block"
                    response = "\n".join(poem_text[:session["current_part"]])
                else:
                    response = "\n".join(
                        poem_text[session["current_part"]: session["current_part"] + 2]
                    )
            else:
                response = "Пожалуйста, повторите строки правильно."

        elif learning_output == "repeat_block":
            correct_text = format_text("".join(poem_text[:session["current_part"]]))
            if user_input == correct_text:
                session["learning_output"] = "repeat_line"
                response = "\n".join(
                    poem_text[session["current_part"]: session["current_part"] + 2]
                )
            elif user_input.lower() in ('повтори', 'повтористрочки', 'повтористроку', 'можешьповторить'):
                response = "\n".join(poem_text[:session["current_part"]])
            else:
                response = "Пожалуйста, повторите строки правильно."

        return {
            "version": "1.0",
            "session": event["session"],
            "response": {
                "text": response,
                "end_session": False,
                "buttons": [{"title": "Выучить другое стихотворение"}] if session_is_end else []
            },
            "session_state": {"session": session},
        }




def handle_request(event):
    """
    Processes the user's request.

     :param event: event
     :return: response for the user
    """
    if "request" in event:
        # Check if the request is a SimpleUtterance and has original_utterance
        if (
            event["request"]["type"] == "SimpleUtterance"
            and "original_utterance" in event["request"]
            and len(event["request"]["original_utterance"]) > 0
        ):
            # Initialize a session dictionary to store state information
            session = {
                "current_part": 0,  # Current part of the poem being learned
                "poem_title": None,  # Title of the poem being learned
                "learning_started": False,  # Flag indicating whether learning has started
                "learning_output": "text",  # Output format for learning
            }

            # Get the title of the poem from the user's utterance
            poem_title = event["request"]["original_utterance"].lower()

            # Retrieve the poem text from the get_poem function
            poem_text = get_poem(poem_title)

            # Check if the poem text was found
            if poem_text is not None:
                # Set the poem title and learning flag
                session["poem_title"] = poem_title
                session["poem_text"] = poem_text
                session["learning_started"] = True

                # Prepare the response
                response = {
                    "version": "1.0",  # Alexa API version
                    "session": session,  # Current session state
                    "response": {
                        "text": "\n".join(poem_text),  # Combine poem parts into a string
                        "buttons": [
                            {
                                # Button to start learning the poem
                                "title": "Начать учить стихотворение",
                                "payload": {"start_learning": True},
                            },
                            {
                                # Button to choice the other poem
                                "title": "Другое стихотворение",
                            },
                        ],
                        "end_session": False,  # Don't end the conversation
                    },
                    "session_state": {"session": session},  # Store session state
                }

                return response

            else:
                # Poem not found, send error message
                response = {
                    "version": "1.0",
                    "session": event["session"],  # Current session
                    "response": {
                        "text": "Стихотворение \"%s\" не найдено.\nПожалуйста, напишите верное название."
                        % poem_title,
                    },
                    "session_state": {"session": session},  # Store session state
                }
                return response
    else:
        # If the request is not a SimpleUtterance, send a welcome message
        return welcome_message(event)





def handler_start(session, event):
    poem_text = session["poem_text"]
    response = "\n".join(
        poem_text[session["current_part"]: session["current_part"] + 2]
    )
    session["learning_output"] = "repeat_line"
    return {
        "version": "1.0",
        "session": event["session"],
        "response": {
            "text": response,
            "end_session": False,
        },
        "session_state": {"session": session},
    }


def handler_other_poem():
    response = {
        "version": "1.0",
        "session": {},
        "response": {"text": "Назови стих, который тебе надо выучить."},
        "session_state": {}
    }
    return response


def handler_help_text():
    response = {
        "version": "1.0",
        "response": {
            "text": "Забыл строчку? Попроси Алису повторить. Ты также можешь выбрать другое стихотворение."
        },
    }
    return response


def handler_can_text():
    response = {
        "version": "1.0",
        "response": {
            "text": "Введи название стихотворения как в примере: Тютчев Ф. И. Листья\nИ повторяй за Алисой.Ты не запомнил строчку?\nПопроси Алису повторить её. Если тебе не понравилось стихотворение, то можешь попросить Алису выбрать другое."
        },
    }
    return response


def handler(event, context):
    """
    Entry-point for Serverless Function.
    :param event: request payload.
    :param context: information about current execution context.
    :return: response to be serialized as JSON.
    """
    text = None
    if 'request' in event and \
            'original_utterance' in event['request'] \
            and len(event['request']['original_utterance']) > 0:
        text = event['request']['original_utterance']
    session = event["state"]["session"].get("session")
    if text is None:
        answer = welcome_message(event)
    elif session and session["learning_started"]:
        if text == "Начать учить стихотворение":
            answer = handler_start(session, event)
        elif text in ("Другое стихотворение", "Выучить другое стихотворение", "выучим другое стихотворение","давай выберем другое стихотворение"):
            answer = handler_other_poem()
        else:
            answer = handle_learning(session, event)
    elif text.lower() in ("помощь", "помоги"):
        answer = handler_help_text()
    elif text.lower() in ("что ты умеешь?", "что ты умеешь"):
        answer = handler_can_text()
    else:
        answer = handle_request(event)

    return answer
