# Single page website creator

![](./assets/default-single-page-website.png)

建立一個 Single Page 網站。

Create a single page website.

**Prompt**

|||
|-------|------|
|**USER**|Make a single page website that shows off different neat javascript features for drop-downs and things to display information. The website should be an HTML file with embedded javascript and CSS.|

**Sample response**

```
<!DOCTYPE html>
<html>
<head>
    <title>JavaScript Features</title>
    <style>
        body {
            font-family: Arial, sans-serif;
        }
        .dropdown {
            position: relative;
            display: inline-block;
        }
        .dropdown-content {
            display: none;
            position: absolute;
            background-color: #f9f9f9;
            min-width: 160px;
            box-shadow: 0px 8px 16px 0px rgba(0,0,0,0.2);
            padding: 12px 16px;
            z-index: 1;
        }
        .dropdown:hover .dropdown-content {
            display: block;
        }
    </style>
</head>
<body>
    <h1>JavaScript Features</h1>
    <div class="dropdown">
        <button>Hover over me</button>
        <div class="dropdown-content">
            <p>Hello World!</p>
        </div>
    </div>

    <h2>JavaScript Alert</h2>
    <button onclick="displayAlert()">Click me</button>

    <h2>JavaScript Date</h2>
    <p id="date"></p>

    <script>
        function displayAlert() {
            alert("Hello, this is a JavaScript Alert!");
        }

        document.getElementById("date").innerHTML = "Today's date is " + new Date().toDateString();
    </script>
</body>
</html> 

This is a simple single-page website that demonstrates a few JavaScript features:

1. A dropdown that appears when you hover over a button. This is achieved using CSS for the hover effect and HTML for the structure.

2. A JavaScript alert that is triggered when you click a button. This is a simple demonstration of how JavaScript can interact with HTML elements.

3. A JavaScript function that displays the current date. This is an example of how JavaScript can manipulate HTML content.
```

**API request**

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
  "model": "gpt-4",
  "messages": [
    {
      "role": "user",
      "content": "Make a single page website that shows off different neat javascript features for drop-downs and things to display information. The website should be an HTML file with embedded javascript and CSS."
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
