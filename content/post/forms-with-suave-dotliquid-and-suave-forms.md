---
title: 'Sending forms with Suave, Suave.DotLiquid and Suave.Forms'
date: '2016-4-21'
categories:
- Suave
tags:
- F#
- Suave 
- Forms
---

In the [previous article](http://vgaltes.com/suave/forms-with-suave-experimental-and-suave-forms) we saw how to send forms using [Suave.Experimental](https://www.nuget.org/packages/Suave.Experimental) as view engine. In this article we're going to use Suave.DotLiquid as view engine. We'll see how we're going to be able to reuse most of the work done previously.

## Suave.DotLiquid ###

[DotLiquid](http://dotliquidmarkup.org) is a port of the Ruby template engine [Liquid](http://dotliquidmarkup.org/). Suave is able to use this library thanks to the pakage [Suave.DotLiquid](https://www.nuget.org/packages/Suave.DotLiquid/). So, the first thing we need to do is install these packages in our project.

## Reusing code ##

We're going to change only the view part of our application, so we dont need to define any new ViewModel or anything. 

## The view ##

To use DotLiquid we need to create temlates as html files. Inside this templates we are able to write constructs that we'll be replace by text. In this simply example we're not going to use anything more sophisticated than this, but you can refer to the [original documentation](https://github.com/dotliquid/dotliquid/wiki/DotLiquid-for-Designers) to see what you can do.

So, the view should look like this:

    <html>
        <head>
            <title>Forms with DotLiquid</title>
        </head>
        <body>
            <div>Create</div>
            <form method="POST">
                <fieldset>
                    <div class="editor-label">Name</div>
                    <div class="editor-field">
                        <input name="Name"type="text"required=""/>
                    </div>
                    <div class="editor-label">Surname</div>
                    <div class="editor-field">
                        <input name="Surname"type="text"required=""/>
                    </div>
                </fieldset>
                <input type="submit"value="Create human"/>
            </form>
        </body>
    </html>
    
In this is standard HTML. Let's take a look at the view that will display the Human model to see DotLiquid in action.

<html>
    <head>
        <title>Forms with DotLiquid</title>
    </head>
    <body>
        <div>Show</div>
        <div>Name: {{model.Name}}</div>
        <div>Surname: {{model.Surname}}</div>
    </body>
</html>

As you can see, we are accessing the properties Name and Surname of the model, in this case the Human type defined in the [previous article]((http://vgaltes.com/suave/forms-with-suave-experimental-and-suave-forms)).

## The application ##

The changes we need to do in our App.fsx are really small.

Firts, let's define the folder where DotLiquid will search for the templates.

    DotLiquid.setTemplatesDir (__SOURCE_DIRECTORY__ + "./Templates")

And now let's define the new WebPart we're going to use.

    let dotliquidHuman =
        choose [
            GET >=>  DotLiquid.page ("CreateHuman.html") ()
            POST >=> bindReq (bindForm Forms.human) (fun form -> DotLiquid.page ("ShowHuman.html") form ) BAD_REQUEST
        ]

As you can see, is very similar to the Experimental one, but we change a little bit how we call the view engine. In this case, we use the function DotLiquid.page, passing as arguments the name of the template and the model to use (unit in the first case, and form in the second case).

And that's all, we don't need to make more changes to make it work.

## Summary ##

We have seen how easy is changing our view engine when working with Forms. We've seen a very little introduction to DotLiquid, a nice template engine we can use with [Suave](https://suave.io). You can get the full solution [here](https://github.com/vgaltes/FSharpForms)