---
title: 'Sending forms with Suave, Suave.Experimental and Suave.Forms'
date: '2016-4-20'
categories:
- Suave
tags:
- F#
- Suave
- Forms
---

Sending data to a web server is a very common task when developing a website. From registering a new user to filling some personal details in a web commerce application, we usually have to deal with filling a form and send its data to the web server. In this series of articles, we are going to see how to send data using different view engines.

## Suave.Experimental ###

Experimental is a component available via [NuGet](https://www.nuget.org/packages/Suave.Experimental) that allows us to write the views using F# code. We are not going to discuss if that's a good thing or not, we are just going to show how to work with forms.

We are not going to explain how to setup a basic [Suave](https://suave.io) application. Plese refer to their [website](https://suave.io) to do it.

## Defining the form ##

Let's start creating a new file called Form.fs with the following code:

    module FSharpForms.Forms

    open Suave.Form

    type Human = {
        Name : string
        Surname : string
    }

    let human : Form<Human> = 
        Form ([ TextProp ((fun f -> <@ f.Name @>), [])
                TextProp ((fun f -> <@ f.Surname @>), [])
                ],
            [])

In this file we are defining the type that we want to be filled by the user, in this case a Human with a Name and a Surname. We are defining a Form of that Human as well, specifying the type of the properties, in this case strings both of them. The empty list that you can see at the end of the definition of each field is a list properties (maxLength, min, max, etc). In this basic text, we don't need any property. The final empty list is a list of ServerSideValidation which is basically a tuple composed by a function that returns an string and a message.

## The view ##

Let's define the view right now. Let's start defining the module and open the necessary modules.

    module FSharpForms.Views

    open System

    open Suave.Html
    open Suave.Form
    open FSharpForms.Forms
    
It's time to define the view. We should write a code like this.

    let createHuman = 
        html [
            head [
                title "Forms with Experimental"
            ]

            body [           
                div [text "Create"]            
                
                tag "form" ["method", "POST"] (flatten 
                    [
                        tag "fieldset" [] (flatten 
                            [
                                divAttr ["class", "editor-label"] [
                                    text "Name"
                                ]
                                divAttr ["class", "editor-field"] [
                                    input (fun f -> <@ f.Name @>) [] Forms.human
                                ]
                                
                                divAttr ["class", "editor-label"] [
                                    text "Surname"
                                ]
                                divAttr ["class", "editor-field"] [
                                    input (fun f -> <@ f.Surname @>) [] Forms.human
                                ]
                            ])
                            
                        inputAttr ["type", "submit"; "value", "Create human"]                
                    ])
            ]
        ]
        |> xmlToString

As you can see we use helper functions to write the html tags. Almost all the tags are part of the Suave.Html module. The only one that belongs to Suave.Form is *input*. Input takes a quotation expression as its first parameter to know which field of the Form is linked to. As last parameter, it takes the form.

We should return an string, so we use *xmlToString* at the end.

## The application ##

Is time to go to our [Suave](https://suave.io) application and make all this code work. Let's start defining the WebPart that we are going to use

    let experimentalHuman =
        choose [
            GET >=>  OK (Views.createHuman)
            POST >=> bindReq (bindForm Forms.human) (fun form -> OK ( Views.showHuman form) ) BAD_REQUEST
        ]

It's a quite simple [WebPart](https://suave.io/routing.html). We're using choose to differentiate between a GET and a POST (we are using the same Url for both actions). In the GET route, we are returning our recently created view. The most interesting part comes in the POST route, where we are binding the information of the POST request to an instance of the Forms.Human type. To display the results we are using the following view:

    let showHuman human = 
        html [
            head [
                title "Forms with Experimental"
            ]

            body [           
                div [text "Show"]
                div [text (sprintf "Name: %s" human.Name)]
                div [text (sprintf "Name: %s" human.Surname)]
        ]
        ]
        |> xmlToString
        
As you can see, we only need to access the fields of the Forms.Human type.

## Summary ##

We have seen an introduction to Forms in Suave. In following articles we'll see how to do the same with other view engines. You can get the full solution [here](https://github.com/vgaltes/FSharpForms)