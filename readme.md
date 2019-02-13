# YelpCamp Likes (campground like button)

### Campground index page (showing the number of likes for each campground):
![YelpCamp Likes - Campgrounds Index Page Screenshot](https://i.imgur.com/PtvBQwd.png)

### Campground show page (like button and the ability to see the users who liked the campground):
![YelpCamp Likes - Campground Show Page Screenshot](https://i.imgur.com/HBcHYaS.png)

- **See more details** button:

![YelpCamp Likes - Campground Show Page Screenshot](https://i.imgur.com/GRP3YmG.png)

This tutorial is based on technologies that we've learned in the course. We will implement logic that will allow us to add the like button functionality to our campgrounds.

### 1) The updated Campground model - see the code here: [models/campground.js](https://github.com/zarkomaslaric/yelpcamp-likes/blob/master/models/campground.js)

First we focus on the Campground model updates that we need to make, in order to store the user likes for each campground.

We create a new array field named `likes` which will hold ObjectId references to the particular users who like the campground (the array will store User model ObjectId references):

```
    likes: [
        {
            type: mongoose.Schema.Types.ObjectId,
            ref: "User"
        }
    ]
```

### 2) Campground routes - see the code here: [routes/campgrounds.js](https://github.com/zarkomaslaric/yelpcamp-likes/blob/master/routes/campgrounds.js)

We need to add a new campground like route (POST route to `/campgrounds/:id/like`) which will handle all the requests to like/unlike a campground. This is how the complete campground like route looks like:

```
// Campground Like Route
router.post("/:id/like", middleware.isLoggedIn, function (req, res) {
    Campground.findById(req.params.id, function (err, foundCampground) {
        if (err) {
            console.log(err);
            return res.redirect("/campgrounds");
        }

        // check if req.user._id exists in foundCampground.likes
        var foundUserLike = foundCampground.likes.some(function (like) {
            return like.equals(req.user._id);
        });

        if (foundUserLike) {
            // user already liked, removing like
            foundCampground.likes.pull(req.user._id);
        } else {
            // adding the new user like
            foundCampground.likes.push(req.user);
        }

        foundCampground.save(function (err) {
            if (err) {
                console.log(err);
                return res.redirect("/campgrounds");
            }
            return res.redirect("/campgrounds/" + foundCampground._id);
        });
    });
});

```

- As mentioned above, it's a POST route with the following path: `/campgrounds/:id/like`

- Since we only want to allow logged in users to like/unlike a campground, we add `middleware.isLoggedIn` to the route

- First we find the specific campground by its id, just like we do in many other routes

- Prior to recording a user like, we want to check the `foundCampground.likes` array to see if a user already liked a campground. We check if the currently logged in user id (req.user._id) exists in the array using the array some() method: [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/some](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/some#Description)

- The some() method will iterate over the `foundCampground.likes` array, calling equals() on each element (ObjectId) to see if it matches req.user._id and stop as soon as it finds a match. If it finds a match it returns true, otherwise it returns false, and we store that boolean value to the `foundUserLike` variable.

- After that, we use the if statement to check the value of `foundUserLike`. If it is true, that means the user already liked a campground, and that they are clicking the **Like** button again in order to remove their like. We use the mongoose pull() method to remove the user ObjectId from the array: [https://mongoosejs.com/docs/api.html#mongoosearray_MongooseArray-pull](https://mongoosejs.com/docs/api.html#mongoosearray_MongooseArray-pull)

- The else clause gets triggered if the user was not found (if they did not like the campground previously), therefore we add the user ObjectId to the `likes` array  using the push() method.

- Finally, we use the save() method to store the campground changes to the database, and redirect the user to the campground show page

##### We also need to populate `likes` for the campground show route:

- Since we are dealing with user ObjectId references in the `likes` array, we need to populate it along the `comments` array in the campground show route. That will allow us to access details of the users who liked the campground (like the username).

```
Campground.findById(req.params.id).populate("comments likes").exec(function (err, foundCampground) {
```

##### Modifications in the campground update (PUT) route:

- I modified the campground PUT route to make sure no unwanted fields get edited (via req.body.campground) when the user is modifying their campground. Check the source code to see the changes: [routes/campgrounds.js](https://github.com/zarkomaslaric/yelpcamp-likes/blob/master/routes/campgrounds.js)

### 3) Changes to header.ejs and footer.ejs: [partials folder link](https://github.com/zarkomaslaric/yelpcamp-likes/tree/master/views/partials)

- **header.ejs** - I added Bootstrap 3.4 (latest version of Bootstrap 3) and the Font Awesome CDN (we are going to be using the thumbs-up icon in our views to represent the like). Be sure to place these above your custom CSS <link>, in the head element:

```
        <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/3.4.0/css/bootstrap.min.css">
        <link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.7.1/css/all.css">
```

- **footer.ejs** - We include CDNs of jQuery and the Bootstrap 3.4 JS because we are going to use the Bootstrap modal (pop-up window) functionality in our campground show page.

```
    <script
            src="https://code.jquery.com/jquery-3.3.1.min.js"
            integrity="sha256-FgpCb/KJQlLNfOu91ta32o/NMZxltwRo8QtmkMRdAu8="
            crossorigin="anonymous"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/3.4.0/js/bootstrap.min.js"
            integrity="sha384-vhJnz1OVIdLktyixHY4Uk3OHEwdQqPppqYR8+5mjsauETgLOcEynD9oPHhhz18Nw"
            crossorigin="anonymous"></script>
    </body>
</html>
```

- You can read more about Bootstrap 3 modals here: [https://getbootstrap.com/docs/3.4/javascript/#modals](https://getbootstrap.com/docs/3.4/javascript/#modals)

### 4) Changes in the campground show EJS template: [views/campgrounds/show.ejs](https://github.com/zarkomaslaric/yelpcamp-likes/tree/master/views/campgrounds/show.ejs)

- This is the main Like button logic that we are adding to the campground show page:

```
<div style="padding-bottom: 10px;">
    <form action="/campgrounds/<%= campground._id %>/like" method="POST">
        <div class="btn-group">
            <% if (currentUser && campground.likes.some(function (like) {
                return like.equals(currentUser._id)
            })) { %>
                <button class="btn btn-sm btn-primary">
                    <i class="fas fa-thumbs-up"></i> Liked (<%= campground.likes.length %>)
                </button>
            <% } else { %>
                <button class="btn btn-sm btn-secondary">
                    <i class="fas fa-thumbs-up"></i> Like (<%= campground.likes.length %>)
                </button>
            <% } %>
            <button type="button" class="btn btn-sm btn-default" data-toggle="modal"
                    data-target="#campgroundLikes">See more details
            </button>
        </div>
    </form>
</div>
```

We are wrapping the button inside a form which sends a POST request to our campground like route.

We use the some() method on the `likes` array again to check if the user already liked the campground, and based on that we create conditional logic inside the EJS tags to display the corresponding button on the page.

To show the total number of likes in our EJS view, we can access the `length` property of the `likes` array:

`<%= campground.likes.length %>`

We also add a button to **See more details**, which opens a Bootstrap modal showing all the users that liked the campground.

* We add the modal logic to the bottom of the show.ejs view:

```
<!-- Campground Likes Modal -->
<div id="campgroundLikes" class="modal fade" role="dialog">
    <div class="modal-dialog">
        <!-- Modal content-->
        <div class="modal-content">
            <div class="modal-header">
                <button type="button" class="close" data-dismiss="modal">&times;</button>
                <h4 class="modal-title">Campground likes: <%= campground.likes.length %></h4>
            </div>
            <div class="modal-body">
                <table class="table table-striped">
                    <thead>
                    <tr>
                        <th>Liked by:</th>
                    </tr>
                    </thead>
                    <tbody>
                    <% campground.likes.forEach(function(like) { %>
                        <tr>
                            <td><span class="badge"><i class="fas fa-user"></i></span> <%= like.username %></td>
                        </tr>
                    <% }); %>
                    <% if (campground.likes.length === 0) { %>
                        <tr>
                            <td><em>No likes yet.</em></td>
                        </tr>
                    <% } %>
                    </tbody>
                </table>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-default" data-dismiss="modal">Close</button>
            </div>
        </div>
    </div>
</div>
```

As mentioned above, you can read more about Bootstrap 3 modals here: [https://getbootstrap.com/docs/3.4/javascript/#modals](https://getbootstrap.com/docs/3.4/javascript/#modals)

* In addition to that, we add a total likes button on the right side of the description segment which also opens the Bootstrap modal to show all the likes:

```
<div class="pull-right">
    <button type="button" class="btn btn-xs btn-primary" data-toggle="modal"
            data-target="#campgroundLikes">
        <span>Total likes: <i class="fas fa-thumbs-up"></i> <%= campground.likes.length %></span>
    </button>
</div>
```

##### Please see the full show.ejs template to overview the changes made: [views/campgrounds/show.ejs](https://github.com/zarkomaslaric/yelpcamp-likes/tree/master/views/campgrounds/show.ejs)

### 5) Changes in the campgrounds index EJS template: [views/campgrounds/index.ejs](https://github.com/zarkomaslaric/yelpcamp-likes/tree/master/views/campgrounds/index.ejs)

* We add the following code inside the existing `<div class="thumbnail">` in our index.ejs template:

```
<div class="thumbnail">
   <img src="<%= campground.image %>">
   <div class="caption">
       <h4><%= campground.name %></h4>
       <div>
           <span class="badge label-primary"><i class="fas fa-thumbs-up"></i> <%= campground.likes.length %></span>
       </div>
   </div>
   <p>
       <a href="/campgrounds/<%= campground._id %>" class="btn btn-primary">More Info</a>
   </p>
</div>
```

# Final words

I hope you'll like the new like campground feature for YelpCamp!

Make sure to check the full repository to review all code changes that needed to be done in order to implement it to our application: [https://github.com/zarkomaslaric/yelpcamp-likes](https://github.com/zarkomaslaric/yelpcamp-likes)
