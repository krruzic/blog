+++
title = "Rocket.rs and Multipart Forms"
date = 2019-12-27
draft = false
type = "post"

[taxonomies]
tags = ["rust", "tutorial", "backend"]
+++

Recently I've been working on a rust web service written using Rocket and Diesel. A big issue I came across was handling
muitpart forms with `content-type: multipart/form-data`. Rocket has yet to add official support for this, but you can
hack it in fairly easily thanks to the Request Guard system.

<!-- more -->

# Setup

Create a new binary project with cargo: `cargo init multipart_demo` then add dependencies to your cargo.toml using
[cargo-edit](https://github.com/killercup/cargo-edit)

```bash
cargo install cargo-edit
cargo add rocket
cargo add rocket-multipart-form-data
cargo add serde_json
# next you have to add serde manually to the cargo file
echo 'serde = { version = "*", features = ["derive"]}' >> Cargo.toml
```

#### Dependencies explained

[Rocket](https://crates.io/crates/rocket): should be obvious if you're here, rocket is the webserver powering the whole
app. It handles incoming requests and routes them to the proper page and does everything in between.

[Rocket-Multipart-Form-Data](https://crates.io/crates/rocket-multipart-form-data): this is where the magic happens. This
package handles the parsing of the multipart form.

[Serde](https://crates.io/crates/serde): JSON serialization/deserialization.

# Getting started

Now that we have our environment set up, it's time to start building something. To keep things simple, let's make a JSON
api endpoint for creating new user profiles. This endpoint should take two fields, `avatar` an image file to use as
their profile picture, and `data` a JSON object containing their name and age.

To do this, create a new file in the `multipart_demo/src` directory called `middleware.rs` this is where we will
intercept the request and use the multipart parser to validate the submission. Instead of just bolting on support
however, let's integrate it with Rocket's powerful
[request guard feature](https://rocket.rs/v0.4/guide/requests/#request-guards).

First we need to set up the Error type in case something goes wrong in parsing, the `User` struct that we deserialize
the JSON to, as well as the `NewUser` struct representing the passed in form.

```rust
// our dependencies
use rocket::data::{FromDataSimple, Outcome};
use rocket::http::Status;
use rocket::{Data, Outcome::*, Request};
use rocket_multipart_form_data::{
    mime, MultipartFormData, MultipartFormDataField,
    MultipartFormDataOptions, RawField, TextField,
};
use serde::{Deserialize, Serialize};

// first we need to create a custom error type, as the FromDataSimple guard
// needs to return one
#[derive(Debug, Clone)]
pub struct MultipartError {
    pub reason: String,
}
impl MultipartError {
    fn new(reason: String) -> MultipartError {
        MultipartError { reason }
    }
}
impl std::fmt::Display for MultipartError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "{}", self.reason)
    }
}

/// simple representation of a user
#[derive(Serialize, Deserialize)]
pub struct User {
    pub name: String,
    pub age: i32,
}

/// multipart form is loaded into this struct
/// this is what's passed through to the route we'll create later
pub struct NewUser {
    /// the submitted image
    pub avatar: Vec<u8>,
    /// we'll deserialize the json into a User
    pub user: User,
}
```

# Request Guard Implementation

Now that we have the basic data structures we need for our app, let's create the parser. As mentioned above, we'll do
this by implementing the Rocket `FromDataSimple` trait as a request guard. What this does is intercept a request BEFORE
it reaches a route, and verifies / modifies / parses the data received. We take advantage of this to verify our
multipart form ahead of time, to keep our routes clean and enable code reuse. Implement the trait below in
`middleware.rs`

```rust
impl FromDataSimple for NewUser {
    type Error = MultipartError;

    fn from_data(request: &Request, data: Data) -> Outcome<Self, Self::Error> {
        let image_bytes;
        let post_obj;
        let mut options = MultipartFormDataOptions::new();

        // setup the multipart parser, this creates a parser
        // that checks for two fields: an image of any mime type
        // and a data field containining json representing a User
        options.allowed_fields.push(
            MultipartFormDataField::raw("avatar")
                .size_limit(8 * 1024 * 1024) // 8 MB
                .content_type_by_string(Some(mime::IMAGE_STAR))
                .unwrap(),
        );
        options
            .allowed_fields
            .push(MultipartFormDataField::text("data").content_type(Some(mime::STAR_STAR)));

        // check if the content type is set properly
        let ct = match request.content_type() {
            Some(ct) => ct,
            _ => {
                return Failure((
                    Status::BadRequest,
                    MultipartError::new(format!(
                        "Incorrect contentType, should be 'multipart/form-data"
                    )),
                ))
            }
        };

        // do the form parsing and return on error
        let multipart_form = match MultipartFormData::parse(&ct, data, options) {
            Ok(m) => m,
            Err(e) => {
                return Failure((Status::BadRequest, MultipartError::new(format!("{:?}", e))))
            }
        };
        // check if the form has the json field `data`
        let post_json_part = match multipart_form.texts.get("data") {
            Some(post_json_part) => post_json_part,
            _ => {
                return Failure((
                    Status::BadRequest,
                    MultipartError::new(format!("Missing field 'data'")),
                ))
            }
        };
        // check if the form has the avatar image
        let image_part: &RawField = match multipart_form.raw.get("avatar") {
            Some(image_part) => image_part,
            _ => {
                return Failure((
                    Status::BadRequest,
                    MultipartError::new(format!("Missing field 'avatar'")),
                ))
            }
        };
        // verify only the data we want is being passed, one text field and one binary
        match post_json_part {
            TextField::Single(text) => {
                let json_string = &text.text.replace('\'', "\"");
                post_obj = match serde_json::from_str::<User>(json_string) {
                    Ok(insert) => insert,
                    Err(e) => {
                        return Failure((
                            Status::BadRequest,
                            MultipartError::new(format!("{:?}", e)),
                        ))
                    }
                };
            }
            TextField::Multiple(_text) => {
                return Failure((
                    Status::BadRequest,
                    MultipartError::new(format!("Extra text fields supplied")),
                ))
            }
        };
        match image_part {
            RawField::Single(raw) => {
                image_bytes = raw.raw.clone();
            }
            RawField::Multiple(_raw) => {
                return Failure((
                    Status::BadRequest,
                    MultipartError::new(format!("Extra image fields supplied")),
                ))
            }
        };
        Success(NewUser {
            user: post_obj,
            avatar: image_bytes,
        })
    }
}
```

This is a lot of code, but it's actually fairly simple. All we do is verify the various parts of the form data, and
return a relevant error message if something goes wrong. If you were verifying a more complex request, you'd just need
to add additional match statements for each field supplied once.

# Finishing Up

Finally we can implement a route that takes in a `NewUser` multipart form! Open up `src/main.rs` and replace its
contents with the below code:

```rust
#![feature(proc_macro_hygiene, decl_macro)]
#[macro_use]
extern crate serde;
#[macro_use]
extern crate rocket;
extern crate rocket_multipart_form_data;

mod middleware;
use crate::middleware::MultipartError;
use crate::middleware::NewUser;

type Result<T> = std::result::Result<T, MultipartError>;

// create a route called create_user that expects a NewUser or an error.
// If the post was successful, print off the user's information, otherwise
// print off the error message.
#[post("/create_user", data = "<multipart>")]
fn new_user(multipart: Result<NewUser>) -> String {
    match multipart {
        Ok(m) => format!("Hello, {} year old named {}!", m.user.age, m.user.name),
        Err(e) => format!("Error: {}", e.reason),
    }
}

fn main() {
    // this launches the webserver with the above route mounted on /.
    // To access it go to localhost:8000/create_user once launched
    rocket::ignite().mount("/", routes![new_user]).launch();
}
```

# Testing our code

See how easy that was? We now have an error-proof route that takes in a multipart form, with no messy code. Since this
is a simple tutorial, I won't show how to make a frontend where you can submit / view / manage users, instead we can
test out this code with good old `curl`. First launch the server with `cargo run`. You should see some output showing
where the route is mounted and how to access it. By default it should be `localhost:8000/create_user`. If for some
reason its different, just replace the references in the curl command. You'll also want to replace `demo.png` with some
png file on your system.

```bash
curl -X POST "localhost:8000/create_user" -H  "accept: */*" -H  \
"Content-Type: multipart/form-data" \
-F data="{'name': 'kristopher', 'age': 25}" -F "avatar=@demo.png;type=image/png"
```

If everything went well, you should see `Hello, 25 year old named kristopher!`. Experiment to see how you can break it!
What happens when you leave out the name field in the JSON? How about when one of the multipart fields is left out
entirely?

[source code](https://github.com/krruzic/multipart-rocket-demo)
