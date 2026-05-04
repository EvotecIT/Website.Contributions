---
title: "Power Apps - How to Work Around the appres://blobmanager issue"
description: "Work around appres://blobmanager issue when saving a file to SharePoint from Power Apps using Power Automate "
date: "2026-05-03"
language: "en"
authors:
  - mateusz-szewczyk
categories:
  - Power Platform
tags:
  - Power Apps
  - Power Automate
  - automation
  - Power Platform
  - SharePoint
image: "./cover.webp"
image_alt: "Diagram showing a Power Apps attachment flowing through Power Automate to SharePoint with an error indicator"
draft: true
---

## Problem description

If you've ever tried to add an attachment control in Power Apps and save the uploaded file to SharePoint using Power Automate, you've probably run into a frustrating problem. When a user selects a file, its value is temporarily stored in an internal Power Apps format as a link starting with `appres://blobmanager/...`. It seems harmless enough, until you actually try to send that file somewhere.
The problem is that Power Automate cannot "fetch" a file from such an address. It is purely an internal temporary identifier that Power Automate has serious trouble interpreting. As a result, the flow either throws an error or an empty and corrupted file ends up on SharePoint.

## Solution

### Step 1. Adding an Attachment Control Without a Form
In Power Apps, you don't need to use a Form component to use the attachment control. Simply copy the YAML code below and paste it into the screen tree. The control will be added automatically.

```yaml
Attachment1:
  Control: Attachments
  Properties:
    BorderColor: =Color.Purple
    BorderThickness: =5
    Fill: =RGBA(169,169,169,.5)
    Height: =268
    MaxAttachments: =1
    Size: =18
    Width: =402
    X: =60
    Y: =111
```

### Step 2. Adding an Attachment and the Value issue
Once the control is added and the user selects a file, we can inspect the value of the first attachment using the formula:

```text
First(Attachment1.Attachments).Value
```
As you can see in the screenshot below, the returned value takes the format appres://blobmanager/... This is an internal temporary address used by Power Apps that cannot be directly passed to Power Automate to save the file to SharePoint.

![Attachment value takes the format appres://blobmanager/](./images/attachment-show-value.png)

### Step 3. Creating a Gallery for File Conversion
To extract the file data in Base64 format, we need to use a gallery. Add a Gallery component to the screen, then in its Items property enter:
```text
Attachment1.Attachments
```
The gallery will now iterate over all added attachments.

![Create gallery and set 'Attachment1.Attachments' as Items property](./images/create-gallery-and-add-items.png)
 
### Step 4. Setting Up the Image Control in the Gallery
Go inside the gallery and select the Image control. In its Value property enter:
```text
ThisItem.Value
```
This way the image control will load the file from the attachment which in the next step will allow us to read its binary data in Base64 format.

![Set 'ThisItem.Value' as image value ](./images/use-value-in-image.png)
 

### Step 5. Hiding the Gallery
The gallery serves purely as a conversion mechanism. The user doesn't need to see it. Set the Visible property of the gallery to:
```text
false
```

### Step 6. Save Button and File Data Collection
Add a button that will trigger the save action. In its OnSelect property, create a collection containing the name and content (in Base64) of each attachment:
```text
ClearCollect(
    colFileContents,
    ForAll(
        Gallery2.AllItems,
        With(
            {
                dataUriSchemePDF:
                    MatchAll(
                        JSON(Image2.Image, JSONFormat.IncludeBinaryData),
                        "(?:[^"":;,]+)"
                    )
            },
            {
                name: Name,
                content: Index(dataUriSchemePDF, 4).FullMatch
            }
        )
    )
)
```
The formula iterates over the gallery items, extracts the binary image data as Base64 using `JSON()` with the IncludeBinaryData flag, then uses `MatchAll()` to parse the resulting string and saves the file name and its Base64 content to the colFileContents collection.


![Use formula to convert appres Value to base64 in on select save button control](./images/save-button-on-select.png)
 

### Step 7. New Flow in Power Automate
Go to Power Automate and create a new flow triggered by a Power Apps trigger. Add one input parameter of type Text named: JSON
In my case the flow is named attachmentsFlow. This name will be needed when calling it from Power Apps.

![Create new flow and use JSON text input](./images/create-flow-input-parameter.png)
 
### Step 8. Apply to Each Loop
Add an Apply to each action. As the list parameter, enter the expression:
```text
json(triggerBody()['text'])
```
Power Automate will parse the submitted JSON and iterate over each file in the collection.

![Add apply to each loop with JSON input](./images/apply-to-each-input.png)


### Step 9. Saving the File to SharePoint
Inside the loop, add a Create file action from the SharePoint connector. Fill in the fields as follows:
-	File Name:
```text
items('Apply_to_each')?['name']
```
-	File Content:
```text
base64ToBinary(items('Apply_to_each')?['content'])
```
The `base64ToBinary()` function converts the Base64 data back into binary form, which SharePoint can save as a file.

![Use 'Create file" to save file in SharePoint with correct name from apply to each and file content from formula base64ToBinary](./images/create-file.png)

### Step 10. Connecting the Flow to Power Apps
Save the flow in Power Automate. Go back to Power Apps, add your flow to the app (the Power Automate tab in the side panel → Add flow), then complete the OnSelect property of the button with the flow call:
```text
attachmentsFlow.Run(JSON(colFileContents))
```
The JSON() function serializes the colFileContents collection into a text JSON format, which the flow will receive as an input parameter and process in the loop.

![Add flow trigger in on select save button with JSON(colFileContents) parameter](./images/save-button-add-flow-trigger.png)

### Step 11. Verifying the Files Were Saved to SharePoint
After launching the app and pressing the save button, we can navigate to the target library on SharePoint and confirm that the files have appeared correctly. As you can see in the screenshot below, the files were saved without any issues.

![Run app, upload attachments and check files in SharePoint library](./images/save-files-check-results.png)


## Summary
As you can see, the appres:// format issue can be worked around without much effort. The key is to indirectly use a gallery and an image control to extract the file data in Base64 format on the Power Apps side. Power Automate then takes care of the rest, decoding the file and saving it to SharePoint. Once configured, the solution works reliably and **works for any file type, not just images**. Despite using an image control as the extraction mechanism, this approach handles any file type: PDFs, Word documents, Excel spreadsheets, ZIP archives, and more. The image control is simply a convenient vehicle for reading appres://blobmanager as Base64. The actual content of the file is preserved intact regardless of its format.

