---
title: "Rocket.rs Snippet for Multipart forms"
date: 2019-07-05T01:29:37-06:00
draft: true
type: "post"
---

Recently I've been working on a rust web service, written using Rocket and Diesel. 
A big issue I came across is handling muitpart forms (AKA `content-type: multipart/form-data`). 
This post briefly goes over how to get it working anyway
<!--more-->
Existing Libraries we'll leverage:
[Multipart](https://crates.io/crates/multipart) is a server agnostic library that ties in multipart forms as a middleware to Iron, Hyper, Nickel and tiny_http. Rocket however is not supported, as the rocket devs want to write support themselves and don't trust this library.
[Rocket-Multipart-Form-Data](https://github.com/magiclen/rocket-multipart-form-data) is a library that uses the built in tools from Multipart to get things working in Rocket.

Now, if you read the README for the second project, you get a pretty good idea on how to jam in support. The problem with this is that it's just tacked on and doesn't integrate with Rocket's powerful features. Most importantly, it doesn't use the `FromData` request guard. 

By implementing this trait, we can intercept a request as it's made, and validate the data being passed. By using this in conjunction with [Rocket-Multipart-Form-Data](https://github.com/magiclen/rocket-multipart-form-data) we can add support to multiple different configurations of `multipart/form-data` as well as handle errors early and cause a Failure in the request before our routes even see it.

Below is an example that looks for a form with two parts: file, a valid image file < 8MB, and json, a JSON object that can be converted to a `Person` struct.

```rust
pub struct Person {
  pub name: String,
  pub age: i32
}

pub struct PostMultipartForm {
    pub image: Vec<u8>,
    pub person: Person,
}


impl FromDataSimple for PostMultipartForm {
    type Error = ();

    fn from_data(request: &Request, data: Data) -> Outcome<Self, Self::Error> {
        let image_bytes;
        let post_obj;
        let mut options = MultipartFormDataOptions::new();

        options.allowed_fields.push(
            MultipartFormDataField::raw("file")
                .size_limit(8 * 1024 * 1024) // 8 MB
                .content_type_by_string(Some(mime::IMAGE_STAR))
                .unwrap(),
        );
        options
            .allowed_fields
            .push(MultipartFormDataField::text("json").content_type(Some(mime::APPLICATION_JSON)));

        // if anything goes wrong unpacking the multipart content
        // return Failure() immediately
        let ct = match request.content_type() {
            Some(ct) => ct,
            _ => return Failure((Status::BadRequest, ())),
        };
        let multipart_form = match MultipartFormData::parse(&ct, data, options) {
            Ok(m) => m,
            Err(_e) => return Failure((Status::BadRequest, OnigiriError::InvalidUpload {})),
        };
        let image_part: &RawField = match multipart_form.raw.get("file") {
            Some(image_part) => image_part,
            _ => return Failure((Status::BadRequest, ())),
        };
        let json_part = match multipart_form.texts.get("json") {
            Some(json_part) => json_part,
            _ => return Failure((Status::BadRequest, ())),
        };

        // verify only the data we want is being passed
        match image_part {
            RawField::Single(raw) => {
                image_bytes = raw.raw.clone();
            }
            RawField::Multiple(_raw) => {
                return Failure((Status::BadRequest, ())),
            }
        };
        match json_part {
            TextField::Single(text) => {
                let json_string = &text.text;
                person_obj = match serde_json::from_str::<Person>(json_string) {
                    Ok(insert) => insert,
                    Err(_e) => {
                        return Failure((Status::BadRequest, ())),
                    }
                };
            }
            TextField::Multiple(_text) => {
                return Failure((Status::BadRequest, ())),
            }
        };

        Success(PostMultipartForm {
            image: image_bytes,
            person: person_obj
        })
    }
}
```

