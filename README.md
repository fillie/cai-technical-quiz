# Cutlure.AI Technical Quiz

I did not use any AI tools during the course of this, bar validating my code on the last question.

I have nothing against using AI tools, and frequently do use them during regular development, but I didn't find any reason or need to to do so. Especially when I was going to then have to potenitally explain it's decisions made in an interview. I have tried to explain all my answers, and how I arrived at them.

In regards to the last question, I did use AI to check the code looked about right, rather than spinning up an entire laravel development environment, as well as helping with some CSS suggestions.

This took me around 3 hours.

#### Question 1

`O:3:"Foo":7:{s:3:"lat";i:50;s:3:"lng";i:53;s:6:"region";s:4:"west";s:3:"bar";s:7:"A
llbama";s:8:"currency";s:3:"USD";s:7:"country";s:13:"United
States";s:4:"test";b:1;}`

This is a **seralised PHP string**. We can tell this is a PHP class or object additionally, from the `O` for object, as **`Foo` is the class name additionally**.

Additionally we can identify the properties contained in the class.

- lat: 50 (integer)
- lng: 53 (integer)
- region: "west" (string)
- bar: "Allbama" (string)
- currency: "USD" (string)
- country: "United States" (string)
- test: true (boolean)

As we can see the **value of `bar` is `Allbama`**, and **`test` is a boolean**, we can tell this because in PHP a `b` denotes a boolean when seralised, and `1` is truthy.

#### Question 2

`eyJsYXQiOjUwLCJsbmciOjUzLjMyNjMsInJlZ2lvbiI6Indlc3QiLCJiYXIiOiJBbGxiYW1hIiwiY3VycmVuY3kiOiJ
VU0QiLCJjb3VudHJ5IjoiVW5pdGVkIFN0YXRlcyIsInRlc3QiOnRydWV9==` is a **base64 encoded string**.

I recognised this by eye, particulary the suffix, and spending too long working with JWTs recently. I verified it by using the following online tool: https://gchq.github.io/CyberChef/.

When base64 decoded, we can see that it is a string of: 

`{"lat":50,"lng":53.3263,"region":"west","bar":"Allbama","currency":"USD","country":"United States","test":true}`

**This string is JSON.** We can also see that **`lng` is equal to: 53.3263`**.

The first thing that came to mind as to why data might be both **JSON, and then base64 encoded is a JWT**, as the standard expects.

#### Question 3

To restore production, I would either **rollback to a previous release** if our deployment pipeline is configured this way, or I would rely on **version pinning to ensure a new deployment is triggered with an older version** of the vendor package we know works.

A few methods spring to mind in regards to fixing this going forwards. Three to highlight:

- **Proper Testing Pipeline:** All changes deployed must first be ran through a test suite which tests at least critical functionality before any broken releases making it to any environment.
- **Proper Environment:** Ensure any changes, including vendor package upgrades, are deployed to another environment first, where we can ensure they work as expected.
- **Version Pinning:** Rather than relying on a package manager to decide what version of a dependency we'd like, pin packages, especially critical and non-dev packages to specific versions we know work. Bonus points if we ensure that our pipeline caches and stores these dependencies should a package be unavailable from a package manager in the future (where upon we can rollback to a package we know works, and we have a copy stored). 
  
#### Question 4

I have had a go at adjusting the code provided, and have summarised the changes I would make.

```
<?php
namespace App;

class MaintenanceManager
{
    private bool $isUp;
    private ?string $reason = null;

    /**
     * Set the system status and reason for downtime.
     *
     * @param bool $isUp
     * @param string|null $reason
     */
    public function setIsUp(bool $isUp, ?string $reason = null): void
    {
        $this->isUp = $isUp;
        $this->reason = $reason;
    }

    /**
     * Return whether the system is up.
     *
     * @return bool
     */
    public function isUp(): bool
    {
        return $this->isUp;
    }

    /**
     * Return whether the system is down.
     *
     * @return bool
     */
    public function isDown(): bool
    {
        return !$this->isUp;
    }

    /**
     * Return the downtime reason.
     *
     * @return string|null
     */
    public function getReason(): ?string
    {
        return $this->reason;
    }
}

class MaintenanceMode
{
    private MaintenanceManager $manager;

    /**
     * Constructor for MaintenanceMode.
     *
     * @param MaintenanceManager $manager
     */
    public function __construct(MaintenanceManager $manager)
    {
        $this->manager = $manager;
    }

    /**
     * Get a formatted downtime message.
     *
     * @return string
     */
    public function getReason(): string
    {
        $downReason = $this->manager->getReason() ?? 'No additional details provided';
        return "Sorry, we are down: " . $downReason;
    }

    /**
     * Check if the system is down.
     *
     * @return bool
     */
    public function isDown(): bool
    {
        return $this->helper->isDown();
    }
}
```

```
<?php
declare(strict_types=1);

use App\MaintenanceManager;
use App\MaintenanceMode;

$manager = new MaintenanceManager();
$manager->setIsUp(
    (bool)($_SERVER['ENV_IS_UP'] ?? true),
    $_SERVER['ENV_DOWN_REASON'] ?? null
);

$mode = new MaintenanceMode($manager);

if ($mode->isDown()) {
    throw new \Exception($mode->getReason());
}

echo file_get_contents('website.html');
```

Changes I've made:

- **Renamed Classes:** Renamed classes from `MaintenanceHelper` to `MaintenanceManager`, in my experience this helps with bload, and ensuring methods and logic, that really belongs elsewhere doesn't get tacked into a general 'helper' class. `MaintenanceMode` is also renamed, to ensure it follows PSR standards. In regards to PSR standards, I have also moved from `<?` to `<?php` additionally.
- **Type Declarations & Strict Types:** - Added type declarations for properties and method returns, helps with type safety, and as a result, should catch bugs, increase readability (and IDE completion), especially across an entire codebase
- **Separation of Concerns:** - Divided the code between managing state and display logic, so there's great separation of the two. In a large project, this is invalulable so we know where to expect certain logic across a codebase, as well as from a readability PoV 
- **Documentation:** - Added docblocks for methods. (I am personally not always a fan of this, I think the method name should be descriptive enough to infer the function, e.g `isDown`, however I realise everyone is different!)
- **Removing `die()`:** - Rather than just halting execution, which is going to be poor UX, adding a chance to throw an exception, and logging an error, allows us to handle this elsewhere in the system and present the user with a proper error screen, as well as assiting with debugging with a proper log

I would also consider using something like `https://github.com/vlucas/phpdotenv` rather than relying on the `$_SERVER` vars as well for better DX, and to handle some of the tpying issues (you'll notice the bool cast on `$_SERVER['ENV_IS_UP']`)

#### Question 5

I have adjusted the schema to as follows, I have attached justifcation:

```
CREATE TABLE users (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL,
    creation_date DATETIME NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uniq_email (email)
);

CREATE TABLE user_settings (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id INT UNSIGNED NOT NULL,
    additional_settings JSON NOT NULL,
    deleted_at TIMESTAMP DEFAULT NULL,
    PRIMARY KEY (id),
    INDEX idx_user_id (user_id),
    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);

CREATE TABLE user_logins (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id INT UNSIGNED NOT NULL,
    ip_address VARCHAR(45) DEFAULT NULL,
    date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    INDEX idx_user_id (user_id),
    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);
```
Changes I've made:

- **Primary Keys:** I have added primary keys as they were missing from the original. Nor were they autoincrementing. Rather than relying on our application layer to be responsible for that, we should offload that responsibility to the DB as that is what it's designed for. Should improve performance if we've been relying on the application doing this work for us (e.g looking up the current largest id + `)
- **Consistent & Better Naming**: Some fields have been using camelCase, where others have been using underscores. I have adopted snake_case (not sure if there's a standard or it's just personal preference!). Additionally I have followed the naming standard from Laravel for timestamps, e.g `created_at` vs `createdDate`. I have also dropped superfluos information, e.g `json` from `other_setting_json` to `additional_settings` (I have made thie field `JSON` which I think is descriptive enough)
- **Improved Field Types:** - As mentioned abvove, I have made `other_setting_json` an actual JSON field, so we can query it if needs be (to play devils advocate, without data, I am unsure if this field should ever actually be JSON).
- **Improved Deletion Handling:** Rather than two deletion fields, we can slim this down to one timestamp field. If the field is filled, then we know the user has been deleted, and the date of so. We don't need both fields to do this.

I also came up with some other considerations during this question which I did not action:

- **Polymorphic Relation**: A general requests table could centralize event logging (like logins and settings changes) but would add complexity, I considered this out of scope for this task.
- **User Deletion**: Instead of automatically deleting login records when a user is removed, preserving them may be beneficial for auditing and debugging, provided the data is GDPR / Legal compliant.```

Additionally, here is a revised query:

```
SELECT users.email, COUNT(user_logins.id) AS totalLogins
FROM users
JOIN user_logins ON user_logins.user_id = users.id
GROUP BY users.id, users.email
HAVING COUNT(user_logins.id) > 3;
```

I have **removed an extra join (`user_settings`)**, as it wasn't actually being used by the query nor the output. Additionally the count is done on the specific field I believe we're interested in. I opted not to use aliases, as I seen value in readability over anything else for the scope of this task.

#### Question 6

There is a fundamental flaw with the provided code, in that there is **mixing of concerns**. Currently, PHP, CSS, JS and PHP (w/ Blade also) are all mixed into the same file, the result is that the flow can be confusing to follow, it is overly complex to read and change and there is a greater chance of introducing issues. As a result, the code is also much harder to unit tests, as logic is mixed within logic.

There also **concerns around security**, and a potential XSS vulnerability in the blade templating used. They have chosen to use `{!! !!}` rather than `{{ }}` which is now displaying values unescaped. There is no reason to do this with an email address.

A few other spots:

- **GET operation is not RESTful**, we are issuing a delete command via a GET operation, contravening REST. We should use DELETE.
- **Inconsistency accessing user data**, in one place we use `$user['email']` and in another `$user->id`. Since this is using Blade, and Laravel, I see no reason not to user the object properties.

A rewrite of this code would ensure that all concerns are separated, and we have distict application layers. In this case I see no reason to deviate from MVC.

I have attempted to clean this up in a reasonable and timely manner, but as with a lot of things in development, I could spend a long time coming up with something perefect. As such I have added some code comments within PHP, where I think it might be worthwhile to point out where I would spend more time in reality.

Controller (**could also be a view composer**, but I see no reason to add more complexity for this task):

```
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function index()
    {
        // Arguably we don't need to return all fields on a user object for this, if this is at scale, I would look to use a select
        return view('users.index', User::all());
    }
    
    public function delete(User $id)
    {
        // I have relied on type hinting the user object and allowing laravel to do the search, with the assumption a middleware would handle the case of a user not being found, otherwise we need to add logic to do so here
        
        // In big systems I would probably consider migrating away from MVC to a service or repository so buisness logic is not handled here, but I have rarely seen use in this at the scale I have worked (but it is good to be aware!)
        $deleted = $user->delete();
        return response()->json(['success' => $deleted]);
    }
}
```

View file using blade:

```
@extends('base')

@section('head')
    <link href="{{ asset('css/user_listing.css') }}" rel="stylesheet" type="text/css" />
@endsection

@section('content')
    <h1>User Listing</h1>
    <div role="main" id="mainBlock" style="--user-count: {{ $users->count() ?: 1 }};">
        @foreach($users as $user)
            <div class="user-block" data-user-id="{{ $user->id }}">
                User: <a href="mailto:{{ $user->email }}">{{ $user->email }}</a>
                Status: Active
            </div>
        @endforeach
    </div>
    <div class="toaster alert w-100" style="display: none;"></div>
@endsection

@section('scripts')
    <script src="{{ asset('js/jquery.js') }}"></script>
    <script src="{{ asset('js/user_listing.js') }}"></script>
    <script>
        window.isTesting = {{ $isTesting ? 'true' : 'false' }};
        window.isAdmin = {{ session('isAdmin') ? 'true' : 'false' }};
    </script>
@endsection
```

JS file:

```document.addEventListener('DOMContentLoaded', function () {
    function showAlert(type, message) {
        const toaster = document.querySelector('.toaster');
        toaster.className = `toaster alert alert-${type}`;
        toaster.textContent = (window.isAdmin === 'true' ? 'admin alert: ' : 'alert: ') + message;
        toaster.style.display = 'block';
    }

    function clickUser(userId) {
        if (window.isTesting === 'true') {
            console.log('Testing mode enabled. No action taken.');
            return;
        }
        if (window.isAdmin !== 'true') {
            showAlert('danger', 'You don\'t have permission to do that');
            return;
        }

        fetch(`/users/${userId}`, {
            method: 'DELETE',
            headers: {
                'Content-Type': 'application/json',
            }
        })
        .then(response => response.json())
        .then(data => {
            if (data.success) {
                showAlert('success', 'User removed');
                const userBlock = document.querySelector(`.user-block[data-user-id="${userId}"]`);
                if (userBlock) {
                    userBlock.remove();
                }
            } else {
                showAlert('danger', 'Failed deleting user');
            }
        })
        .catch(error => {
            showAlert('danger', error.message || 'An error occurred');
        });
    }

    document.querySelectorAll('.user-block').forEach(function(block) {
        block.addEventListener('click', function() {
            const userId = this.getAttribute('data-user-id');
            clickUser(userId);
        });
    });
});
```
