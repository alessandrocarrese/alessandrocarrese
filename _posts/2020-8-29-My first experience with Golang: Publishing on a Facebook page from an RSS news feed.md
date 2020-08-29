---
layout: post
title: My first experience with Golang: Publishing on a Facebook page from an RSS news feed.
---

I've always been suspicious about golang even if the web is full of people talking about it. So I decided to try it for a simple project.
I wanted to build a small code to publish on a facebook page the items fetched from a news feed (in particular Google RSS News Feed).
I did it in Go, and I admit it has been a surprise to see how fast the development has been. I have a bit of experience with C# and I love Elixir as functional language, but I mostly use Python so coming from Python syntax
the simplicity of the Go's syntax for me was not really interesting, and I did not quite like it before developing this project.
It felt strange to me to declare a variable using the ":=" symbols, or importing modules withous using the comma, and start the for loop that looks like C based one but without the needing of parenthesis.
Anyway I loved the way the compiler works and the ability of building the code to an executable avoiding all the burdains of installing libraries on other machines.
Finally I must say: I like GO.

# The project

To publish on a Facebook page it is necessary to have the right permissions from Facebook and a Facebook App

**First Step: Create a Facebook App**

from https://developers.facebook.com/apps

**Second step: Create a Facebook Short Lived User Access Token**

To create a short lived user token we can use:
- Graph Explorer tool
- Facebook Login Dialog 

I did it using the Graph Explorer Tool choosing the app previously created.

Type of token: 
- user 

Permissions:
- pages_show_list
- pages_read_engagement
- pages_manage_posts
- public_profile

**Third step: Generate a long lived user access (optional)**
To generate a long lived token we need to make a GET request with the app_id, app_secret and the short lived acccess token previously generated:

    curl -i -X GET "https://graph.facebook.com/oauth/access_token?grant_type=fb_exchange_token&
    client_id={app-id}&
    client_secret={app-secret}&
    fb_exchange_token={short-lived-user-access-token}"
This token has 60 days expiration days, but we don't care since we are going to use it only as requirement to generate a non-expiring page token;

**Fourth step: Generate a page token**
Here we need to make another GET with the token we've jsut obtained. To do this we also need the id of the page we want to manage from API (to find this there are a lot of small guides online).

    curl -i -X GET "https://graph.facebook.com/{page-id}?
    fields=access_token&
    access_token={user-access-token}"
  
**Fifth Step: The code**

To publish the feed it is necessary to parse it and to send the API call to the facebook endpoint.
To parse the feed I used "gofeed", installed via "go get github.com/mmcdole/gofeed"; 
To make the api request I used "nrt/http";

This is the code I mixed:
package main


    import ("fmt"
    "github.com/mmcdole/gofeed"
    "net/http"
    "net/url"
    "strconv"
    "strings"
    "log"
    "io/ioutil"
    )

    func main() {
	     page_token := "here the page token"
	     fp := gofeed.NewParser()
	
	     feed, _ := fp.ParseURL("https://news.google.com/rss?hl=it&gl=IT&ceid=IT:it")
	     for i:=0; i<len(feed.Items); i++{
		     fmt.Println(feed.Items[i].Title)
		     data := url.Values{}
		     data.Set("message", feed.Items[i].Title)
		     data.Set("link", feed.Items[i].Link)
		     data.Set("access_token", page_token)
		     url := "https://graph.facebook.com/{here the facebook page id}/feed"
		     client := &http.Client{}
		     r, err := http.NewRequest("POST", url, strings.NewReader(data.Encode())) // URL-encoded payload
		     if err != nil {
			     log.Fatal(err)
		    }
		    r.Header.Add("Content-Type", "application/x-www-form-urlencoded")
		    r.Header.Add("Content-Length", strconv.Itoa(len(data.Encode())))

		     res, err := client.Do(r)
		     if err != nil {
			     log.Fatal(err)
		     }
		     log.Println(res.Status)
		     defer res.Body.Close()
		
		     body, err := ioutil.ReadAll(res.Body)
		     if err != nil {
			    log.Fatal(err)
		     }
		
		      log.Println(string(body))
	     }

     }

Note: this code fetch the feed, iterate over it and create a POST using the Publish a Link endpoint from Facebook:

     curl -i -X "POST https://graph.facebook.com/{page-id}/feed
       ?message=Smart video calling to fit every family
       &link=https://portal.facebook.com/products/
       &access_token={page-access-token}"
