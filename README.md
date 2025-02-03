GENERATED IN PARTNERSHIP WITH AI 

Listen to the accompanying audio on Spotify:  
[Click here to listen](https://spotifycreators-web.app.link/e/QJzbuxSXFQb)

# Background

BAML is a cutting-edge domain-specific language designed to seamlessly integrate the semantic prowess of large language models (LLMs) with the precision of traditional deterministic programming. By generating structured outputs that are reliable and developer-friendly, BAML addresses the common challenges of non-deterministic text generation inherent in many LLM applications. Rather than relying on repetitive prompts to achieve valid results, BAML provides a robust framework that ensures consistency and accuracy, making it an indispensable tool for developers who need to merge the flexibility of natural language understanding with the rigidity of structured data processing.

This document assumes that you are using python and you have installed BAML on Visual Studio. 


# Introduction 
## Hello World
BAML's core abstraction is a function that written in a type-safe domain-specific language (DSL). The BAML toolchain addresses the problem of consistent and reliable structured generation with LLMs by implementing a compiler and a parser.  The BAML toolchain allows you to transpile BAML into native Python, Typescript, Ruby, etc code. The BAML parser obtains the LLM’s output and applies post facto fixes and validation to the output. Instead of relying on costly methods like re-prompting the LLM to fix minor issues in the outputs (which takes seconds), the BAML engine corrects the output in milliseconds, thus saving money on API calls while also allowing you to use smaller, cheaper models that can achieve largely the same outcome as bigger, more expensive models.

------------------------------------------ helloworld of BAML ------------------------------------

Here's the most simple function that illustrates the core abstraction: 

```
function Calculate(str: string) -> string {
  client CustomGPT4oMini

  prompt #"
    {{str}}
    {{ctx.output_format}}
  "#
}
```

In this function, we are expecting a return value of string (normal chat completion from nearly all LLMs) with the input of a string (str).  

If the input to the function Calculate is "if x = 1 and y = 3, what is the value of x + y? ", we expect that somewhere in the description , we will get the answer of 4.  But that 4 can be anywhere. How can we improve through BAML? 

```
class RetValue {
  value int
}
function Calculate(str: string) -> RetValue {
  client CustomGPT4oMini

  prompt #"
    {{str}}
    {{ctx.output_format}}
  "#
}

```
------------------------------------------ end helloworld of BAML ------------------------------------
## clients 
The clients are defined in clients.baml

Specifically 
```
client<llm> CustomGPT4oMini {
  provider openai
  retry_policy Exponential
  options {
    model "gpt-4o-mini"
    api_key env.OPENAI_API_KEY
  }
}
You can define multiple clients 

client<llm> CustomO3 {
  provider openai
  options {
    model "o3-mini"
    api_key env.OPENAI_API_KEY
  }
}

But also more importantly, you can group these client at a higher level
client<llm> CustomRoundRobin {
  provider round-robin
  options {
    // This will alternate between the two clients
    strategy [CustomGPT4oMini, CustomO3]
  }
}
if customRoundRobin is used in Calculate such as here: 
function Calculate(str: string) -> RetValue {
  client CustomRoundRobin

  prompt #"
    {{str}}
    {{ctx.output_format}}
  "#
}

```
then 
```
from baml_client import b

str = input("Please input something for the llm to calculate")

print(b.Calculate(str=str))
print(b.Calculate(str=str))
``` 
will use multiple LLMs in a round robin fashion


## Types 
You can define the types through enums or classes
### Basic Typing


#### Enums

##### Basic 
Let’s look at a domain in finance to see how enums capture a set of known, fixed values. Imagine we have an order processing system where each payment goes through distinct statuses. Instead of representing these statuses as strings (which can be error-prone and less descriptive), we can use an enum to enumerate all the valid states. This approach provides better type safety and clarity in our code.

Below is an example of how you might define an enum for PaymentStatus in BAML:

-------------------------------------------------------

```
enum PaymentStatus {
  PENDING @description(#"
    The payment has been initiated, but the final status has not yet been determined.
  "#)

  COMPLETED @description(#"
    The payment was successfully processed.
  "#)

  FAILED @description(#"
    The payment could not be processed due to an error.
  "#)

  REFUNDED @description(#"
    The payment was refunded to the customer.
  "#)
}
```
-------------------------------------------------------

In this example, the enum PaymentStatus defines four possible values: PENDING, COMPLETED, FAILED, and REFUNDED. Each value is accompanied by an inline description using the @description annotation. These descriptions serve as documentation that explains the meaning of each status, which can be incredibly useful for developers and for creating user-friendly error or status messages in an interface.

Now let’s create a function that makes use of this enum. Suppose we have a function that determines the status of a payment based on some input criteria. We might define it as follows:

-------------------------------------------------------
```
function DeterminePaymentStatus(details: string) -> PaymentStatus {
  client CustomRoundRobin

  prompt #"
    Based on the following payment details, determine the status of the payment.
    ---------------------------------------------------
    {{ details }}
    {{ ctx.output_format }}
  "#
}
```

-------------------------------------------------------
##### Dynamic Enums
Let's consider a customer support system where new types of support issues emerge over time. As your support team grows and the product evolves, you might start receiving issues that don't neatly fall into your original fixed categories. In this scenario, a dynamic enum lets you adjust the classification of support tickets at runtime.

Imagine you initially define a dynamic enum for ticket categories like so:

-------------------------------------------------------
```
enum TicketCategory {
  BUG          // Issues where something isn’t working as expected
  FEATURE      // Requests for new enhancements or capabilities
  INQUIRY      // General questions or requests for information
  @@dynamic    // This dynamic marker allows new categories to be added later
}
```
-------------------------------------------------------

You then create a function that analyzes the ticket description and suggests a category based on its contents:

-------------------------------------------------------

```
function CategorizeTicket(description: string) -> TicketCategory {
  client CustomRoundRobin

  prompt #"
    Analyze the following customer support ticket and classify it as either BUG, FEATURE, INQUIRY, or an appropriate category if it doesn’t fit the defaults. 
    ---------------------------------------------------
    {{ description }}
    {{ ctx.output_format }}
  "#
}
```
-------------------------------------------------------

In practice, as your product and support needs change, you might encounter a ticket that doesn’t easily match one of the original categories. For example, suppose customers start reporting an issue that seems related to performance under heavy load—a category not initially considered. With dynamic enums, you can update your TicketCategory at runtime without modifying the original DSL code.

Here’s an example in Python that shows how you might add a new category ("PERFORMANCE") using the BAML TypeBuilder:

```
-------------------------------------------------------
from baml_client.type_builder import TypeBuilder
from baml_client import b

async def run():
    # Create a TypeBuilder instance to modify dynamic enums.
    tb = TypeBuilder()
    
    # Add a new runtime category not originally specified.
    tb.TicketCategory.add_value('PERFORMANCE')
    
    # Now pass the typebuilder as part of the baml_options when calling the function.
    description = "The system experiences significant slowdowns during peak times."
    result = await b.CategorizeTicket(description, { "tb": tb })
    
    # The result can now be one of BUG, FEATURE, INQUIRY, or PERFORMANCE.
    print("Ticket categorized as:", result)
-------------------------------------------------------
```

#### Classes
##### Dynamic Classes

Here's a simple example illustrating the concept of dynamism

```
class User {
  name string
  age int
  @@dynamic
}
function DynamicUserCreator(user_info: string) -> User {
  client GPT4
  prompt #"
    Extract the information from this chunk of text:
    "{{ user_info }}"
     
    {{ ctx.output_format }}
  "#
}
```


```
from baml_client.type_builder import TypeBuilder
from baml_client import b
async def run():
  tb = TypeBuilder()
  tb.User.add_property('email', tb.string())
  tb.User.add_property('address', tb.string()).description("The user's address")
  res = await b.DynamicUserCreator("some user info", { "tb": tb })
  # Now res can have email and address fields
  print(res)

```


Imagine you’re building an inventory system for a car dealership. Vehicles in your inventory may have many properties (such as brand, model, year, engine type, fuel type, color, etc.), and over time you might need to support additional attributes—say, electric range for electric cars or towing capacity for trucks—without rewriting the original class definitions.

Below is a BAML DSL example for a dynamic Vehicle class and a function to extract vehicle information from a raw description:

-------------------------------------------------------
```
class Vehicle {
  brand  string
  model  string
  year   int
  @@dynamic
}

function ExtractVehicleInfo(description: string) -> Vehicle {
  client CustomRoundRobin

  prompt #"
    You are provided with a description of a vehicle. Extract its details and output them in the following JSON format:
    
    {{ ctx.output_format }}
    
    Description:
    ---------------------------------------------------
    {{ description }}
  "#
}
```
-------------------------------------------------------

In this example, we define a Vehicle class with basic properties, and the @@dynamic directive indicates that it can be extended at runtime. Suppose you start receiving vehicle data that includes additional attributes like fuel type and electric range for hybrid or electric vehicles. You can use the BAML TypeBuilder in Python to enrich the Vehicle class at runtime:

-------------------------------------------------------

```
from baml_client.type_builder import TypeBuilder
from baml_client import b

async def run():
    # Create a TypeBuilder instance to extend dynamic classes.
    tb = TypeBuilder()
    
    # Dynamically add new properties to Vehicle.
    tb.Vehicle.add_property('fuel_type', tb.string()).description("The type of fuel used by the vehicle (e.g., gasoline, diesel, electric).")
    tb.Vehicle.add_property('electric_range', tb.int()).description("The electric range of the vehicle in miles, if applicable.")
    
    # Example vehicle description that might contain additional details.
    description = (
        "This 2022 model is a sleek sedan from BrandX. It runs on hybrid power with an electric range of 45 miles and uses gasoline as backup."
    )
    
    # Call the function with the dynamic properties provided through the type builder.
    result = await b.ExtractVehicleInfo(description, { "tb": tb })
    
    print("Extracted vehicle information:")
    print(result)
```
-------------------------------------------------------
### Advanced Typing

#### Creating new dynamic classes or enums not in BAML

Imagine you’re building an online marketplace. Initially, the BAML definitions include a basic User type containing only name and age. Later, your product team decides that users should also have a preferred membership tier (like “Gold,” “Silver,” or “Bronze”) and a detailed address for shipping purposes. However, the original DSL doesn’t have definitions for MembershipTier or Address. Instead of rewriting your DSL code, you can leverage BAML’s TypeBuilder to “inject” these new types dynamically at runtime.


────────────────────────────────────────
Step 1. Create New Dynamic Types

You start by creating a new dynamic enum for MembershipTier. This enum wasn’t defined beforehand in the DSL. With TypeBuilder you add the enum “MembershipTier” and populate it with the possible values, for example, Gold, Silver, and Bronze. Then, you create a new dynamic class “Address” to capture a user’s shipping details (like street, city, and postal code).

Example code:
────────────────────────────────────────

```
from baml_client.type_builder import TypeBuilder
from baml_client.async_client import b

async def run():
  # Create a TypeBuilder instance to manage all dynamic types.
  tb = TypeBuilder()
  
  # Create a new dynamic enum for membership tiers.
  membership_enum = tb.add_enum("MembershipTier")
  membership_enum.add_value("Gold")
  membership_enum.add_value("Silver")
  membership_enum.add_value("Bronze")

  # Create a new dynamic class for the Address.
  address_class = tb.add_class("Address")
  address_class.add_property("street", tb.string()).description("The street address where the user lives")
  address_class.add_property("city", tb.string()).description("The city where the user lives")
  address_class.add_property("postal_code", tb.string()).description("The postal code for the address")

  # Extend the existing User type by adding properties for membership and address.
  # Both are optional, so if the data isn’t present, the function can still succeed.
  tb.User.add_property("membership_tier", membership_enum.type().optional())\
         .description("The user's preferred membership tier for special discounts and perks")
  tb.User.add_property("address", address_class.type().optional())\
         .description("The detailed address information for user shipping purposes")
  
  # Use the extended User type via a function that extracts user info from a text blob.
  # Note: DynamicUserCreator is a BAML function that originally expected only name and age.
  res = await b.DynamicUserCreator("User: Jane Doe, age 34, prefers Gold benefits, lives at 123 Market St, Metropolis, 12345", {"tb": tb})
  
  print("Extracted user information:")
  print(res)

────────────────────────────────────────
```
#### BAML Type System

BAML's type system now supports a comprehensive set of features, designed specifically for language models:

Primitives: string, float, int, bool, literal, null
Composite: map<k, v>, list<v>
User-defined: classes, enums

You can literally define any type in BAML (including arbitrary JSON).
type JSON = string | int | float | null | JSON[] | map<string, JSON>;


##### Recursive types

Recursive types allow a type to refer to itself (directly or indirectly). This is useful when you need to model nested or hierarchical structures. For example, consider an employee in an organization who can have subordinates that are themselves employees. You can define an Employee class that “recursively” contains a list of Employee objects.
```

class Employee {
  name      string
  
  subordinates Employee[] @description(#"
    People reporting to employee
  "#)
}


function DetermineTeamStructure (str: string) -> Employee {
  client CustomO3
  prompt #"
    Extract reporting structure out of text
    ---------------------------------------
    {{ str }}
    ---------------------------------------
    
    {{ ctx.output_format }} 
  "#
}
```


With sufficient advanced LLMs (o3-mini, deepseekr1), with the followin input text

"George is the CEO of the company. Kelly is the VP of Sales. Asif is the global head of product development. Mohammed manages the shopping cart experience. Tim manages sales in South. Stefan is responsible for sales in the f100 company. Carol is in charge of user experience", 

the recursive data structure of 
```
name='George' subordinates=[Employee(name='Kelly', subordinates=[Employee(name='Tim', subordinates=[]), Employee(name='Stefan', subordinates=[])]), Employee(name='Asif', subordinates=[Employee(name='Mohammed', subordinates=[]), Employee(name='Carol', subordinates=[])])]
```

can be extracted. 


## Client Registry

### Introduction 

Below is a simple, real-world use case that shows why and how you might want to use a Client Registry in BAML.

Imagine you’re building a resume screening tool. You’ve already written a BAML function called ExtractResume that processes a candidate’s resume using an LLM client defined in your .baml files. Initially, your function might look like this:

-------------------------------------------------------

```
function ExtractResume(text: string) -> Resume {
  client DefaultLLMClient

  prompt #"
    Extract the candidate's key details (name, experience, skills, etc.) from the resume text below.
    ---------------------------------------------------
    {{ text }}
    {{ ctx.output_format }}
  "#
}

```
-------------------------------------------------------

At some point during development you notice two important things:

1. You’d like to tweak the LLM parameters (for example, using a different model or a lower temperature to ensure more deterministic output) for better quality extraction.
2. You want to be able to test different configurations (for example, switching between GPT-4o and a fallback LLM) based on runtime conditions without modifying the DSL code directly.

This is where the Client Registry comes in handy. Using the ClientRegistry at runtime, you can override or replace the client settings for your function. For example, you can add a new client that uses a specific model and parameters and then set it as primary so that your ExtractResume function uses it for the API calls.

Here’s a simplified Python example using the ClientRegistry:

-------------------------------------------------------

```
import os
from baml_py import ClientRegistry
from baml_client import b  # Assume b is our transpiled BAML module with ExtractResume

async def run():
    # Create a new client registry instance.
    cr = ClientRegistry()

    # Add a new client configuration (this overrides any pre-defined client with the same name).
    # In this case, we're configuring our client to use "gpt-4o" for more deterministic output.
    cr.add_llm_client(
        name='MyAmazingClient',
        provider='openai',
        options={
            "model": "gpt-4o",
            "temperature": 0.7,
            "api_key": os.environ.get('OPENAI_API_KEY')
        }
    )

    # Set 'MyAmazingClient' as the primary client.
    cr.set_primary('MyAmazingClient')

    # Now, when you call ExtractResume, it will use MyAmazingClient instead of the default client.
    resume_text = "Jane Doe, 5 years of data analysis experience, proficient in Python and SQL..."
    result = await b.ExtractResume(resume_text, { "client_registry": cr })

    print("Extracted Resume Information:")
    print(result)
```

When run, this function uses the client settings provided in the client registry,
allowing you to adjust parameters easily at runtime without touching the DSL.
-------------------------------------------------------

Using the Client Registry in this way has several advantages:

• Flexibility: You can change the LLM model, adjust temperature, or update other parameters dynamically based on runtime conditions or user preferences.

• Overriding Defaults: Even if you defined a default client in your .baml files, you can easily override it with specific configurations when needed.

• Experimentation: Test different models or strategies (such as the round-robin strategy shown earlier) for handling various types of resumes without modifying your BAML DSL code.

This simple use case shows how client registry empowers you to manage and modify LLM client configurations on the fly—saving time, reducing costs, and improving the overall reliability of your LLM-powered applications.




### Advanced Use Case for Client Registry

Consider the use case that leverages the advanced client registry to both monitor the consistency and performance of several LLM backends. In this example, imagine you’re building a critical math “evaluation” service that uses a BAML‐defined function (named Calculate) to compute simple math expressions. Because your customers rely on accurate and fast evaluations, you want to validate not only that each available LLM client returns the correct answer but also record each client’s response time and error resiliency before deciding which to assign to different tiers of your service.

In this use case, you have several LLM clients (for example, o3-mini, gpt-4o-mini, deepseekr1, llama3370b8192) already registered in a YAML file. At runtime, you iterate over each client, use the ClientRegistry to set that client as primary temporarily, then call your BAML function Calculate with a simple arithmetic prompt. You check whether the returned value is correct and also log the response time. This process enables you to track which LLM yields the right answers fastest or under specific conditions, so that you can later offer “performance” or “cost-optimized” tiers based on measured resiliency.

The following Python code illustrates this alternative approach:

------------------------------------------------------------

```
#!/usr/bin/env python3
import asyncio
import time
import yaml
import os
import inspect

# Import ClientRegistry and the transpiled BAML module (assumed to be available as b)
from baml_py import ClientRegistry
from baml_client import b  # b contains our transpiled functions (e.g., Calculate)
from baml_client import sync_client   # if needed for introspection

# An async utility to test a function using a given client
async def evaluate_with_client(cr: ClientRegistry, prompt: str, client_name: str, expected: int):
    # Set the given client as primary
    cr.set_primary(client_name)
    start = time.perf_counter()
    try:
        # Call the BAML function (e.g., Calculate), passing the client_registry override.
        # (Note: Our Calculate function is assumed to have a signature like:
        #  function Calculate(str: string) -> RetValue {...},
        #  where RetValue has an integer field named "value".)
        result = await b.Calculate(prompt, {"client_registry": cr})
        duration = time.perf_counter() - start
        # Evaluate if the function returned the expected result.
        is_correct = (result.value == expected)
        print(f"Client: {client_name} | Response Time: {duration*1000:.1f}ms | Correct: {is_correct} | Result: {result.value}")
        return (client_name, duration, is_correct)
    except Exception as ex:
        duration = time.perf_counter() - start
        print(f"Client: {client_name} | ERROR after {duration*1000:.1f}ms: {ex}")
        return (client_name, duration, False)

async def main():
    # Initialize a new ClientRegistry
    cr = ClientRegistry()

    # Load client configurations from a YAML file.
    # Assume clients.yaml has structure similar to:
    #   CustomGPT4oMini:
    #     provider: openai
    #     options:
    #       model: "gpt-4o-mini"
    #       api_key: "OPENAI_API_KEY"
    #   CustomO3:
    #     provider: openai
    #     options:
    #       model: "o3-mini"
    #       api_key: "OPENAI_API_KEY"
    #
    with open("clients.yaml", "r") as file:
        client_data = yaml.safe_load(file)

    # Add each client from the YAML into the ClientRegistry.
    for client_name, client_info in client_data.items():
        provider = client_info['provider']
        # Allow the API key to be resolved from an environment variable.
        options = client_info['options']
        if "api_key" in options:
            env_key = options["api_key"]
            options["api_key"] = os.environ.get(env_key)
        cr.add_llm_client(client_name, provider=provider, options=options)

    # The math expression under test.
    # For example, "if x = 1 and y = 1, what is x+y?"
    # (Here we expect 2 as the answer, but note the sample above mentioned 4 for different input.)
    prompt = "if x = 1 and y = 1, what is x+y?"
    expected_answer = 2
    
    # Option 1: Iterate over each client defined in our registry by reading the YAML keys.
    print("\n--- Testing Each Client Individually ---\n")
    test_results = []
    for client_name in client_data.keys():
        result = await evaluate_with_client(cr, prompt, client_name, expected_answer)
        test_results.append(result)

    # Option 2: Dynamically iterate over all BAML functions (if you have more than one to evaluate)
    # For demonstration, here we only test the Calculate function.
    methods = inspect.getmembers(sync_client.BamlSyncClient, inspect.isfunction)
    instance_methods = [(name, func)
                        for name, func in methods
                        if not (isinstance(func, classmethod) or isinstance(func, staticmethod))]
    print("\n--- Available BAML Functions in sync_client ---")
    for name, func in instance_methods:
        print(f"Function Name: {name}")

    # You could extend this loop to evaluate multiple functions against each client.
    
    # After testing, you may choose which client(s) to set as primary for production.
    # For example, if CustomGPT4oMini was fastest and correct consistently, you can register it as primary:
    chosen_client = min(test_results, key=lambda x: x[1] if x[2] else float('inf'))[0]
    cr.set_primary(chosen_client)
    print(f"\n=> Setting {chosen_client} as the primary production client based on above evaluations.")

# Run the async evaluation
if __name__ == "__main__":
    asyncio.run(main())

```
------------------------------------------------------------


This approach allows you to systematically evaluate multiple LLM configurations in real time so that you can balance cost and performance while ensuring that fallback strategies or A/B tests are data-driven.












