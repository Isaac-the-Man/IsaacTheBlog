---
title: "Get Structured Output with Function Calling from Google's Gemini"
date: 2024-05-23T14:23:02+08:00
draft: false
author: "Yu-Kai \"Steven\" Wang"
tags: ["LLM", "Gemini", "prompt", "gen AI"]
categories: ["machine learning"]
featuredImage: '/images/robot_learning_to_talk.png'
---

The other day I was working on generating a synthetic dataset using an LLM to evaluate another model that my company is currently developing. 
The workflow involves asking the LLM to read an article and extract some useful keywords from what it reads. 
I was using Google's *Gemini 1.0 Pro* for the task and it understood the question fairly well, the only problem left is to find a way to format the model's response in a structured way (like a JSON) for the response to be used later.

For the simplicity of this demo, we'll ask the LLM to classify a list of organisms by whether they know how to fly or not and try to convert the response into a structured format.

### Method 1 - Ask Politely...?
Let's say we want the LLM to generate the response in JSON format. Since the model understands *natural language* and probably what a JSON is too, we can simply ask it to give us the output in JSON.

```python
# initialize Gemini Model
vertexai.init(project=PROJECT_ID, location=LOCATION)
model = GenerativeModel("gemini-1.0-pro-001")
chat = model.start_chat()

# prompt engineering
prompt = Content(
    role="user",
    parts=[
        Part.from_text("""
            import vertexai
            from vertexai.generative_models import GenerativeModel, Tool, FunctionDeclaration, Content, Part
            Here's a list of organisms:
            1. Tree Frog
            2. Giraffe
            3. Hammerhead Shark
            4. Praying Mantis
            6. Penguins
            7. Flying squirrel
            8. Bat
            9. Ostrich
            10. Sea gull
            For each living things mentioned above, tell me if it flies or not.
            Return the response as a JSON object with the key being the name of the organism, 
            and the value being the True/False (boolean) of wether it flies or not.
        """)
    ]
)

# get response from model
response = chat.send_message(
    prompt,
    generation_config={"temperature": 0},
)
print(response.text)
```

We'll set the temperature to 0, making the LLM less creative and more predictable. 

Raw response below:

````
```
{
  "Tree Frog": false,
  "Giraffe": false,
  "Hammerhead Shark": false,
  "Praying Mantis": false,
  "Penguins": false,
  "Flying squirrel": true,
  "Bat": true,
  "Ostrich": false,
  "Sea gull": true
}
```
````

The model does the job surprisingly well. It is able to understand the schema of the JSON I wanted with just a few lines of verbal description. 
The only imperfection about this response (except for the fact that the praying mantis does know how to fly) is the markdown backticks surrounding the actual JSON syntax.
Normally this doesn't pose much of a problem unless this is part of an automation script that requires the model output to be strictly JSON. 

*But what if we need a more complicated schema?*

Let's try adding another field "habitat" to each of the creatures and see how the model responds.

```python
# initialize Gemini Model
vertexai.init(project=PROJECT_ID, location=LOCATION)
model = GenerativeModel("gemini-1.0-pro-001")
chat = model.start_chat()

# prompt engineering
prompt = Content(
    role="user",
    parts=[
        Part.from_text("""
            Here's a list of organisms:
            1. Tree Frog
            2. Giraffe
            3. Hammerhead Shark
            4. Praying Mantis
            6. Penguins
            7. Flying squirrel
            8. Bat
            9. Ostrich
            10. Sea gull
            For each living things mentioned above, tell me if it flies or not.
            Return the response as a JSON object with the key being the name of the organism, 
            and the value being a JSON object with the key "isFly" with values set to True/False (boolean) of wether it flies or not,
            and another key "habitat" with values explaning the natural habitat of such creature. 
        """)
    ]
)

# get response from model
response = chat.send_message(
    prompt,
    generation_config={"temperature": 0},
)
print(response.text)
```

See raw response below: 

````
```json
{
  "Tree Frog": {
    "isFly": false,
    "habitat": "Rainforests and other humid environments"
  },
  "Giraffe": {
    "isFly": false,
    "habitat": "African savannas and woodlands"
  },
  "Hammerhead Shark": {
    "isFly": false,
    "habitat": "Coastal waters and estuaries"
  },
  "Praying Mantis": {
    "isFly": false,
    "habitat": "Gardens, fields, and other open areas"
  },
  "Penguins": {
    "isFly": false,
    "habitat": "Southern Hemisphere oceans"
  },
  "Flying squirrel": {
    "isFly": true,
    "habitat": "Forests"
  },
  "Bat": {
    "isFly": true,
    "habitat": "Caves, trees, and other dark places"
  },
  "Ostrich": {
    "isFly": false,
    "habitat": "African savannas"
  },
  "Sea gull": {
    "isFly": true,
    "habitat": "Coastal areas"
  }
}
```
````

*Still not a problem for the model!*

However, the verbal description of the schema is getting a little bit complicated to read already.
Also notice how the markdown syntax at line 1 is slightly different from the last response.

*But what if we need an even more complicated schema?*

This time we'll add a list of primary food sources and their estimated calories.

```python
# initialize Gemini Model
vertexai.init(project=PROJECT_ID, location=LOCATION)
model = GenerativeModel("gemini-1.0-pro-001")
chat = model.start_chat()

# prompt engineering
prompt = Content(
    role="user",
    parts=[
        Part.from_text("""
            Here's a list of organisms:
            1. Tree Frog
            2. Giraffe
            3. Hammerhead Shark
            4. Praying Mantis
            6. Penguins
            7. Flying squirrel
            8. Bat
            9. Ostrich
            10. Sea gull
            For each living things mentioned above, tell me if it flies or not.
            Return the response as a JSON object with the key being the name of the organism, 
            and the value being a JSON object with the key "isFly" with values set to True/False (boolean) of wether it flies or not,
            and another key "habitat" with values explaning the natural habitat of such creature,
            and another key "food", with values being a list of JSON object that represents the primary food sources for that specific creature.
            Each of the primary food source JSON object should have the key "name" which is the name of the primary food source, 
            and another key "calories" which indicates the approximate calories of each unit of that food.
        """)
    ]
)

# get response from model
response = chat.send_message(
    prompt,
    generation_config={"temperature": 0},
)
print(response.text)
```

See the raw response below:

````
```JSON
{
  "Tree Frog": {
    "isFly": false,
    "habitat": "Tropical and temperate forests",
    "food": [
      {
        "name": "Insects",
        "calories": 10
      },
      {
        "name": "Small vertebrates",
        "calories": 50
      }
    ]
  },
  "Giraffe": {
    "isFly": false,
    "habitat": "African savannas and woodlands",
    "food": [
      {
        "name": "Leaves",
        "calories": 15
      },
      {
        "name": "Twigs",
        "calories": 10
      },
      {
        "name": "Fruits",
        "calories": 20
      }
    ]
  },
  "Hammerhead Shark": {
    "isFly": false,
    "habitat": "Coastal waters and open oceans",
    "food": [
      {
        "name": "Fish",
        "calories": 100
      },
      {
        "name": "Squid",
        "calories": 80
      },
      {
        "name": "Crustaceans",
        "calories": 60
      }
    ]
  },
  "Praying Mantis": {
    "isFly": false,
    "habitat": "Gardens, fields, and forests",
    "food": [
      {
        "name": "Insects",
        "calories": 10
      },
      {
        "name": "Small vertebrates",
        "calories": 50
      }
    ]
  },
  "Penguins": {
    "isFly": false,
    "habitat": "Coastal waters and islands of the Southern Hemisphere",
    "food": [
      {
        "name": "Fish",
        "calories": 100
      },
      {
        "name": "Squid",
        "calories": 80
      },
      {
        "name": "Krill",
        "calories": 60
      }
    ]
  },
  "Flying squirrel": {
    "isFly": true,
    "habitat": "Forests",
    "food": [
      {
        "name": "Nuts",
        "calories": 20
      },
      {
        "name": "Seeds",
        "calories": 15
      },
      {
        "name": "Fruits",
        "calories": 20
      }
    ]
  },
  "Bat": {
    "isFly": true,
    "habitat": "Caves, trees, and buildings",
    "food": [
      {
        "name": "Insects",
        "calories": 10
      },
      {
        "name": "Fruit",
        "calories": 20
      },
      {
        "name": "Nectar",
        "calories": 15
      }
    ]
  },
  "Ostrich": {
    "isFly": false,
    "habitat": "African savannas and woodlands",
    "food": [
      {
        "name": "Plants",
        "calories": 15
      },
      {
        "name": "Seeds",
        "calories": 10
      },
      {
        "name": "Insects",
        "calories": 10
      }
    ]
  },
  "Sea gull": {
    "isFly": true,
    "habitat": "Coastal areas and inland waterways",
    "food": [
      {
        "name": "Fish",
        "calories": 100
      },
      {
        "name": "Squid",
        "calories": 80
      },
      {
        "name": "Crustaceans",
        "calories": 60
      }
    ]
  }
}
```
````

*Perfectly executed!*

Putting the accuracy of the information aside, this is a pretty impressive feat. 
Still, the description of this schema is getting out of hand. 
If only we have a more expressive way to describe the model schema output.


### Method 2 - Function Calling

*Function calling* offers an easy way for developers to specify the input of an API call as a pre-defined template for the model to use so that when the model feels like calling the specific API endpoint, it will format its output as specified for the user to make the API call.

For example, let's declare a function endpoint that is supposed to return the weather forecast of a specific location (lines 6 ~ 24).
Note that we don't have to actually implement this API functionality. We are simply defining the parameters required to call this API for the model, 
using a template-like syntax.

Next, we'll have to use a prompt that invokes the need to call this function (line 31) and provide the model with this new function declaration for it to use (line 40).

```python
from vertexai.generative_models import GenerativeModel, Tool, FunctionDeclaration, Content, Part
from google.cloud.aiplatform_v1beta1.types.tool import FunctionCall

model = GenerativeModel(MODEL)

# declare function
get_weather_by_location = FunctionDeclaration(
    name="get_weather",
    description="Get weather forecast given a location.",
    parameters={
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "Location of interest (city/region)."
            },
            "range": {
                "type": "integer",
                "description": "Days to forecast."
            }
        },
        "required": ["location", "range"]
    }
)

# prompt definition
user_prompt_content = Content(
    role="user",
    parts=[
        Part.from_text("""
            What is the weather forecast for the next 7 days in Taipei?
        """)
    ]
)

# call to Gemini
res = model.generate_content(
    user_prompt_content,
    generation_config={"temperature": 0},
    tools=[Tool(function_declarations=[get_weather_by_location])]
)

# extract function call output
print(FunctionCall.to_dict(res.candidates[0].function_calls[0])['args'])
```

See raw output below:

```
{'range': 7.0, 'location': 'Taipei'}
```

*perfect!*

By using *function calling*, we can get a structured output with a simple and clean syntax. No more nested schema descriptions!
The output also doesn't include the markdown wrapper since it strictly follows whatever schema we've defined in the function declaration.

*But how do we apply function calling to format the output of a response?*

We can rewrite the previous examples of flying creatures as a function declaration. 
We'll have to then phrase the function (as in line 9 below) and prompt (line 48) in a way that tricks the model into using this function, 
forcing it to format its output according to the schema we defined. 
Here we claim that our function formats the response into JSON, and then asks the model to respond in JSON.

```python
from vertexai.generative_models import GenerativeModel, Tool, FunctionDeclaration, Content, Part
from google.cloud.aiplatform_v1beta1.types.tool import FunctionCall

model = GenerativeModel(MODEL)

# declare function
format_creature_bio_json = FunctionDeclaration(
    name="format_creature_bio",
    description="Format a creatures' bio to JSON format.",
    parameters={
        "type": "object",
        "properties": {
            "isFly": {
                "type": "boolean",
                "description": "Indicates whether the creature flies."
            },
            "habitat": {
                "type": "string",
                "description": "Natural habitat of the creature."
            },
            "food": {
                "type": "array",
                "description": "list of primary food sources of the creature.",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {
                            "type": "string",
                            "description": "name of the food source."
                        },
                        "calories": {
                            "type": "integer",
                            "description": "approximate calories of each unit of the food source."
                        }
                    }
                }
            }
        },
        "required": ["isFly", "habitat", "food"]
    }
)

# prompt definition
user_prompt_content = Content(
    role="user",
    parts=[
        Part.from_text("""
            Tell me if a tree frog flies, what is its habitat, and its primary food sources. Return it in JSON.
        """)
    ]
)

# call to Gemini
res = model.generate_content(
    user_prompt_content,
    generation_config={"temperature": 0},
    tools=[Tool(function_declarations=[format_creature_bio_json])]
)

# extract function call output
print(FunctionCall.to_dict(res.candidates[0].function_calls[0])['args'])
```

See raw response below:

````
{'food': [{'calories': 100.0, 'name': 'insects'}], 'habitat': 'rainforests', 'isFly': False}
````

Here we only asked the model about the creature "Tree Frog", and it responded with the exact format template as we described. 
**Function calling is usually more robust compared to simply asking the model to output a schema we described verbally in the prompt.**

However, one of the downsides of using function calling is that there can be no dynamic fields/keys in the structured output. 
This means that we'll have to make a function call for each of the creatures on the list, and merge the data together later, 
since function calling prohibits the model from outputting an object with unknown keys (name of the creature in our case) until runtime.

```python
model = GenerativeModel(MODEL)

creatures = [
    'Tree Frog',
    'Giraffe',
    'Hammerhead Shark',
    'Praying Mantis',
    'Penguins',
    'Flying squirrel',
    'Bat',
    'Ostrich',
    'Sea gull'
]

# declare function
format_creature_bio_json = FunctionDeclaration(
    name="format_creature_bio",
    description="Format a creatures' bio to JSON format.",
    parameters={
        "type": "object",
        "properties": {
            "isFly": {
                "type": "boolean",
                "description": "Indicates whether the creature flies."
            },
            "habitat": {
                "type": "string",
                "description": "Natural habitat of the creature."
            },
            "food": {
                "type": "array",
                "description": "list of primary food sources of the creature.",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {
                            "type": "string",
                            "description": "name of the food source."
                        },
                        "calories": {
                            "type": "integer",
                            "description": "approximate calories of each unit of the food source."
                        }
                    }
                }
            }
        },
        "required": ["isFly", "habitat", "food"]
    }
)

# make a call for each creature
data = {}
for c in creatures:

    # prompt definition
    user_prompt_content = Content(
        role="user",
        parts=[
            Part.from_text("""
                Tell me if a {} flies, what is its habitat, and its primary food sources. Return it in JSON.
            """.format(c))
        ]
    )

    # call to Gemini
    res = model.generate_content(
        user_prompt_content,
        generation_config={"temperature": 0},
        tools=[Tool(function_declarations=[format_creature_bio_json])]
    )

    # extract function call output
    data[c] = FunctionCall.to_dict(res.candidates[0].function_calls[0])['args']

print(data)
```

See raw response below:

```
{'Bat': {'food': [{'calories': 100.0, 'name': 'insects'}],
         'habitat': 'caves',
         'isFly': True},
 'Flying squirrel': {'food': [{'calories': 100.0, 'name': 'seeds'},
                              {'calories': 200.0, 'name': 'nuts'}],
                     'habitat': 'forests',
                     'isFly': True},
 'Giraffe': {'food': [{'calories': 100.0, 'name': 'Acacia leaves'}],
             'habitat': 'African savanna',
             'isFly': False},
 'Hammerhead Shark': {'food': [{'calories': 100.0, 'name': 'fish'},
                               {'calories': 200.0, 'name': 'squid'}],
                      'habitat': 'ocean',
                      'isFly': False},
 'Ostrich': {'food': [{'calories': 100.0, 'name': 'plants'}],
             'habitat': 'savannah',
             'isFly': False},
 'Penguins': {'food': [{'calories': 100.0, 'name': 'krill'},
                       {'calories': 200.0, 'name': 'fish'}],
              'habitat': 'ocean',
              'isFly': False},
 'Praying Mantis': {'food': [{'calories': 100.0, 'name': 'insects'}],
                    'habitat': 'grasslands',
                    'isFly': False},
 'Sea gull': {'food': [{'calories': 100.0, 'name': 'Fish'},
                       {'calories': 200.0, 'name': 'Bread'}],
              'habitat': 'Coastal areas',
              'isFly': True},
 'Tree Frog': {'food': [{'calories': 100.0, 'name': 'insects'},
                        {'calories': 200.0, 'name': 'worms'}],
               'habitat': 'rainforests',
               'isFly': False}}
```

That's it! 

As a rule of thumb, you will likely want to use function calling whenever possible, but if your schema isn't all that complex, a verbal description in the prompt works just fine.

### References
1. [Function calling](https://ai.google.dev/gemini-api/docs/function-calling)
2. [Function calling with the Gemini API](https://ai.google.dev/gemini-api/docs/function-calling/tutorial?lang=python)