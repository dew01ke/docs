## Invoke the function {#invoke}

{% note info %}

To allow any user to invoke your function, [make it public](../../functions/operations/function/function-public.md). For more information about access rights, see [{#T}](../../functions/security/index.md).

{% endnote %}

{% list tabs %}

- Management console

   1. In the [management console]({{ link-console-main }}), select the folder containing your function.
   1. Select **{{ sf-name }}**.
   1. Select a function.
   1. Go to the **Testing** tab.
   1. In the **Tag version** field, specify ```$latest``` to invoke the latest function version.
   1. Under **Payload template**, choose **Without template**.
   1. In the **Input** field, enter:
      ```
      	{"queryStringParameters": {"name": "Username"}}
      ```
   1. Click **Run test**.
   1. You will see the testing status under **Test result** in the **Function status** field. **Important**: Maximum function execution time before [timeout](../../functions/operations/function/version-manage.md) (including original initialization at first call) is 10 minutes.
   1. You will see the function execution result in the **Function output** field:
      ```
      	{
      		"statusCode": 200,
      		"headers": {
      			"Content-Type": "text/plain"
      		},
      		"isBase64Encoded": false,
      		"body": "Hello, Username!"
      	}
      ```

- CLI

   {% include [cli-install](../cli-install.md) %}

   {% include [default-catalogue](../default-catalogue.md) %}

   To invoke a function, run the command:

   ```
   yc serverless function invoke <function ID> -d '{"queryStringParameters": {"name": "Username"}}'
   ```

   Result:

   ```
   {"statusCode":200,"headers":{"Content-Type":"text/plain"},"isBase64Encoded":false,"body":"Hello, Username!"}
   ```

   The function version with the `$latest` tag is invoked by default.

- HTTPS

   You can view the function invocation link on the **Overview** tab, in the **Link to invoke** field.

   For security reasons, you can only invoke the function via HTTPS. Invoke it as a regular HTTP request by inserting the function invocation link in the browser address bar and adding the `name` parameter to the URL:

   ```
   	{{ sf-url }}/b09bhaokchn9pnbrlseb?name=Username
   ```

   The following response appears on the page:

   ```
   	Hello, Username!
   ```

- {{ yandex-cloud }} Toolkit

   You can call a function using the [{{ yandex-cloud }} Toolkit plugin](https://github.com/yandex-cloud/ide-plugin-jetbrains/blob/master/README.en.md) for the IDE family on the [IntelliJ platform](https://www.jetbrains.com/opensource/idea/) from [JetBrains](https://www.jetbrains.com/).

{% endlist %}
