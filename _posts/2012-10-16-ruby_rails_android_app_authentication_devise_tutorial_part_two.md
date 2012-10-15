---
layout: post
title: Ruby on Rails and Android Authentication Part Two
tagline: The Android App
category: tutorial
tags: [ruby on rails, android, devise, authentication]
---
{% include JB/setup %}

In the [first part of the tutorial](/tutorial/2012/10/15/ruby_rails_android_app_authentication_devise_tutorial_part_one), you have been guided through the coding of a complete Ruby on Rails backend that can be used to register and login users through a JSON API.

In the second part then, you'll learn how to code an Android native app that will consume that API. This tutorial doesn't want to start from scratch about Android development, so I advice you to take a look at this series of tutorials from [mobile.tutsplus.com](http://mobile.tutsplus.com/articles/news/learn-android-sdk-development-from-scratch) that will guide you during the first steps of setting up the Eclipse environment.

Once you have Eclipse up and running with everything you need, we can start coding our app.

### App creation and setup
Open up Eclipse and navigate the menus to create a new Android application: <code>"File > New > Other ... > Android Application Project"</code>

Choose the Application/Project/Package names appropriately and select these SDKs:

- Build SDK: Android 4.1 (API 16)
- Minimum required SDK: API 8 Android 2.2 (Froyo)

We target the latest Android version and we support devices down to Froyo.

Choose if you want to create a Launcher Icon or not and create a blank activity and call it <code>HomeActivity</code>.

## HomeActivity

### Dummy Tasks controller
In order to implement the JSON authentication, I wanted to first test the api client on our Android app. To do so I added a "fake" tasks controller that renders an arbitrary JSON response. I leave you the actual work to implement something more "real" and dynamic.

{% highlight ruby %}
# file: app/controllers/api/v1/tasks_controller.rb
class Api::V1::TasksController < ApplicationController
  skip_before_filter :verify_authenticity_token,
                     :if => Proc.new { |c| c.request.format == 'application/json' }

  # Just skip the authentication for now
  # before_filter :authenticate_user!

  respond_to :json

  def index
    render :text => '{
  "success":true,
  "info":"ok",
  "data":{
          "tasks":[
                    {"title":"Complete the app"},
                    {"title":"Complete the tutorial"}
                  ]
         }
}'
  end
end
{% endhighlight %}

Don't forget to add the route under the API::V1 namespace.

{% highlight ruby %}
# file: config/routes.rb
namespace :api do
  namespace :v1 do
    devise_scope :user do
      post 'registrations' => 'registrations#create', :as => 'register'
      post 'sessions' => 'sessions#create', :as => 'login'
      delete 'sessions' => 'sessions#destroy', :as => 'logout'
    end
    get 'tasks' => 'tasks#index', :as => 'tasks'
  end
end
{% endhighlight %}

You can test this new resource at the address [http://localhost:3000/api/v1/tasks.json](http://localhost:3000/api/v1/tasks.json)

### Reading JSON asynchronously with UrlJsonAsyncTask
The HomeActivity will be the core of our simple app, it retrieves the tasks saved on the Rails backend and show them as a list.

To read the JSON from the application, without interrupting the UI thread, we will use an helper (with some additional classes) written by Tony Lukasavage ([savagelook.com](http://savagelook.com)) called <code>UrlJsonAsyncTask</code>.

To add this nice set of tools, just download the package from Github ([github.com/tonylukasavage/com.savagelook.android](https://github.com/tonylukasavage/com.savagelook.android)) and drag and drop the unzipped "com" directory and its content into the Package Explorer in Eclipse. If you did it right, a <code>com.savagelook.android</code> new package will appear under the "src" directory.

In the following snippet of code I will show you the code I added to the HomeActivity that Eclipse automatically wrote when we created our app.

In order to access our local Rails server from the Android Emulator, we need to use the <code>10.0.2.2</code> IP address that the Emulator maps to the localhost on our machine. Keep in mind to change this constant to the actual address (and port) you will use in production.

The <code>onCreate()</code> method, after the usual initialization, calls the <code>loadTasksFromAPI()</code> where we create and initialize the <code>GetTasksTask</code> (I know ...) that extends the <code>UrlJsonAsyncTask</code> class.

The important thing to understand about this useful helper is that we need to override the <code>onPostExecute()</code> method, telling how to use the JSON we hopefully received from the backend.

In this case we just extract the tasks array and setup the adapter with the list of task's titles found.

{% highlight java %}
// file: HomeActivity.java
private static final String TASKS_URL = "http://10.0.2.2:3000/api/v1/tasks.json";

@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_home);

    loadTasksFromAPI(TASKS_URL);
}

private void loadTasksFromAPI(String url) {
    GetTasksTask getTasksTask = new GetTasksTask(HomeActivity.this);
    getTasksTask.setMessageLoading("Loading tasks...");
    getTasksTask.execute(url);
}

private class GetTasksTask extends UrlJsonAsyncTask {
    public GetTasksTask(Context context) {
        super(context);
    }

    @Override
        protected void onPostExecute(JSONObject json) {
            try {
                JSONArray jsonTasks = json.getJSONObject("data").getJSONArray("tasks");
                int length = jsonTasks.length();
                List<String> tasksTitles = new ArrayList<String>(length);

                for (int i = 0; i < length; i++) {
                    tasksTitles.add(jsonTasks.getJSONObject(i).getString("title"));
                }

                ListView tasksListView = (ListView) findViewById (R.id.tasks_list_view);
                if (tasksListView != null) {
                    tasksListView.setAdapter(new ArrayAdapter<String>(HomeActivity.this,
                      android.R.layout.simple_list_item_1, tasksTitles));
                }
            } catch (Exception e) {
            Toast.makeText(context, e.getMessage(),
                Toast.LENGTH_LONG).show();
        } finally {
            super.onPostExecute(json);
        }
    }
}
{% endhighlight %}

Major information about UrlJsonAsyncTask can be found at [buildmobile.com/waiting-for-android-urljsonasynctask](http://buildmobile.com/waiting-for-android-urljsonasynctask).

Here's the simple layout with the ListView ready to display our tasks.

{% highlight xml %}
<!-- file: res/layout/activity_home.xml -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >
    <ListView android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/tasks_list_view"></ListView>
</LinearLayout>
{% endhighlight %}

Further information on the ListView can be found here: http://developer.android.com/guide/topics/ui/layout/listview.html

It's important to add the permission to use the internet connection in the Manifest: without it our app couldn't reach the API, even on our localhost machine and running the emulator.

{% highlight xml %}
<!-- file: AndroidManifest.xml -->
<uses-permission android:name="android.permission.INTERNET"/>
{% endhighlight %}

### Bonus: Menus
I wanted to add a "Refresh" action to our activity so I added these two methods (the first should be already there at the creation) to setup the menu/actionbar and respond to clicks on their items.

{% highlight java %}
// file: HomeActivity.java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.activity_home, menu);
    return true;
}

@Override
public boolean onOptionsItemSelected(MenuItem item) {
    // Handle item selection
    switch (item.getItemId()) {
        case R.id.menu_refresh:
            loadTasksFromAPI(TASKS_URL);
            return true;
        default:
            return super.onOptionsItemSelected(item);
    }
}
{% endhighlight %}

A default file called <code>res/menu/activity_home.xml</code> should be found, edit it accordantly to the above code with the right id and strings.

Clicking on refresh should now retrieve the tasks list again.

## WelcomeActivity
Before adding the functionality to authorize the access to our tasks to an authenticated user, we want to prompt him to register or login.

To do so, we will change the HomeActivity to check if the user already obtained an authorization token from our Rails backend.

We'll do it by adding another override method <code>onResume()</code>:

{% highlight java %}
// file: HomeActivity.java
private SharedPreferences mPreferences;

@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_home);

    mPreferences = getSharedPreferences("CurrentUser", MODE_PRIVATE);
}

@Override
public void onResume() {
    super.onResume();

    if (mPreferences.contains("AuthToken")) {
        loadTasksFromAPI(TASKS_URL);
    } else {
        Intent intent = new Intent(HomeActivity.this, WelcomeActivity.class);
        startActivityForResult(intent, 0);
    }
}
{% endhighlight %}

As you can see, we moved the <code>loadTasksFromAPI()</code> call from the <code>onCreate()</code> method to the newly added one.
The <code>mPreferences</code> variable has been added and we will use the <code>SharedPreferences</code> class to store our login informations later on. More information about this interface can be found here: [developer.android.com/guide/topics/data/data-storage.html#pref](http://developer.android.com/guide/topics/data/data-storage.html#pref)

What we are going to do now with this code, is to check whether or not there's an auth_token set for our user. If it's present, just load the tasks; if not launch the <code>WelcomeActivity</code>.

To add a new activity, click on the menu <code>"File > New > Other ... > Android activity"</code> and select a blank one and call it <code>"WelcomeActivity"</code>.

This activity will be really simple, just presenting a message and two buttons: Register and Login.

{% highlight java %}
// file: WelcomeActivity.java
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_welcome);

    findViewById(R.id.registerButton).setOnClickListener(
        new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                // No account, load new account view
                Intent intent = new Intent(WelcomeActivity.this,
                    RegisterActivity.class);
                startActivityForResult(intent, 0);
            }
        });

    findViewById(R.id.loginButton).setOnClickListener(
        new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                // Existing Account, load login view
                Intent intent = new Intent(WelcomeActivity.this,
                    LoginActivity.class);
                startActivityForResult(intent, 0);
            }
        });
}

@Override
public void onBackPressed() {
    Intent startMain = new Intent(Intent.ACTION_MAIN);
    startMain.addCategory(Intent.CATEGORY_HOME);
    startMain.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    startActivity(startMain);
    finish();
}
{% endhighlight %}

Inside the <code>onCreate()</code> method we simply set the layout and add two onClickListener to the buttons: to start the RegisterActivity from the Register button and to start the LoginActivity from the Login button.

I also added another override - <code>onBackPressed()</code> - because I want to consider this activity as the "main" one until the user has been logged in. So whenever he or she wants to exit our app without having registered/signed in, he or she will be sent back to the Home screen of the phone without going back to the HomeActivity. In other words, he or she won't see the HomeActivity until logged in: the WelcomeActivity will be considered as the actual main activity.

Here's the simple layout for our WelcomeActivity:

{% highlight xml %}
<!-- file: res/layout/activity_welcome.xml -->
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="15dp" >

    <TextView
        android:id="@+id/welcomeTitle"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@string/welcome_title"
        android:textAppearance="?android:attr/textAppearanceLarge" />

    <TextView
        android:id="@+id/welcomeText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@string/welcome_text"
        android:textAppearance="?android:attr/textAppearanceSmall" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="20dp"
        android:orientation="horizontal" >

        <Button
            android:id="@+id/registerButton"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="@string/register" />

        <Button
            android:id="@+id/loginButton"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="@string/login" />
    </LinearLayout>

</LinearLayout>
{% endhighlight %}

Don't forget to add the needed strings to the <code>res/values/strings.xml</code> file.

Finally, register the new activity in the <code>AndroidManifest.xml</code> file:

{% highlight xml %}
<!-- file: AndroidManifest.xml -->
<activity
    android:name=".WelcomeActivity"
    android:label="@string/title_activity_welcome"
    android:noHistory="true" >
</activity>
{% endhighlight %}

I added the <code>android:noHistory</code> attribute because I don't want to allow the user to return to this activity once he or she successfully logged in.

## LoginActivity
Right now the app won't start because we didn't create the LoginActivity and RegisterActivity yet. Let's proceed with the usual method we already covered for the WelcomeActivity.

Once we have the LoginActivity file created, let's start with the layout file:

{% highlight xml %}
<!-- file: res/layout/activity_login.xml -->
<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="15dp" >

        <EditText
            android:id="@+id/userEmail"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:ems="10"
            android:hint="@string/user_email"
            android:inputType="textEmailAddress" >

            <requestFocus />
        </EditText>

        <EditText
            android:id="@+id/userPassword"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:ems="10"
            android:hint="@string/user_password"
            android:inputType="textPassword" />

        <Button
            android:id="@+id/loginButton"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:onClick="login"
            android:text="@string/login" />
    </LinearLayout>

</ScrollView>
{% endhighlight %}

As you can see we added two EditText fields and a Button to a ScrollView, in order to allow the controls to show no matter what the screen size/rotation.

Let's now put a good use of this interface. Open the <code>LoginActivity.java</code> file and add this variables and methods.

{% highlight java %}
// file: LoginActivity.java
private final static String LOGIN_API_ENDPOINT_URL = "http://10.0.2.2:3000/api/v1/sessions.json";
private SharedPreferences mPreferences;
private String mUserEmail;
private String mUserPassword;

@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_login);

    mPreferences = getSharedPreferences("CurrentUser", MODE_PRIVATE);
}
{% endhighlight %}

If you recall the <code>activity_login.xml</code> layout file, we added an <code>android:onClick</code> attribute to the Button, with a <code>login</code> value. This is the method <code>login()</code> we have to add to our Activity that will be called when the user click on the button.


{% highlight java %}
// file: LoginActivity.java
public void login(View button) {
    EditText userEmailField = (EditText) findViewById(R.id.userEmail);
    mUserEmail = userEmailField.getText().toString();
    EditText userPasswordField = (EditText) findViewById(R.id.userPassword);
    mUserPassword = userPasswordField.getText().toString();

    if (mUserEmail.length() == 0 || mUserPassword.length() == 0) {
        // input fields are empty
        Toast.makeText(this, "Please complete all the fields",
            Toast.LENGTH_LONG).show();
        return;
    } else {
        LoginTask loginTask = new LoginTask(LoginActivity.this);
        loginTask.setMessageLoading("Logging in...");
        loginTask.execute(LOGIN_API_ENDPOINT_URL);
    }
}
{% endhighlight %}

The method just collect the values from the two EditText fields and save the user's email and password into our two variables, <code>mUserEmail</code> and <code>mUserPassword</code>.
We then check if they are not empty strings and alert the users if they are, prompting him or her to provide all the information to login in.

If everything seems ok, we create a new <code>LoginTask</code> to execute the actual authentication with our Rails backend.

The <code>LoginTask</code> is another class we extend from <code>UrlJsonAsyncTask</code> just like we did in the <code>HomeActivity</code> to load the tasks from the API.
This time, though, we will override not only the <code>onPostExecute()</code> method but also the <code> doInBackground()</code> one.
We have to do this because the standard implementation doesn't allow us to post data to the JSON API, but it's really easy to do otherwise.

Add this new class inside the <code>LoginActivity</code> class.

{% highlight java %}
// file: LoginActivity.java
private class LoginTask extends UrlJsonAsyncTask {
    public LoginTask(Context context) {
        super(context);
    }

    @Override
    protected JSONObject doInBackground(String... urls) {
        DefaultHttpClient client = new DefaultHttpClient();
        HttpPost post = new HttpPost(urls[0]);
        JSONObject holder = new JSONObject();
        JSONObject userObj = new JSONObject();
        String response = null;
        JSONObject json = new JSONObject();

        try {
            try {
                // setup the returned values in case
                // something goes wrong
                json.put("success", false);
                json.put("info", "Something went wrong. Retry!");
                // add the user email and password to
                // the params
                userObj.put("email", mUserEmail);
                userObj.put("password", mUserPassword);
                holder.put("user", userObj);
                StringEntity se = new StringEntity(holder.toString());
                post.setEntity(se);

                // setup the request headers
                post.setHeader("Accept", "application/json");
                post.setHeader("Content-Type", "application/json");

                ResponseHandler<String> responseHandler = new BasicResponseHandler();
                response = client.execute(post, responseHandler);
                json = new JSONObject(response);

            } catch (HttpResponseException e) {
                e.printStackTrace();
                Log.e("ClientProtocol", "" + e);
                json.put("info", "Email and/or password are invalid. Retry!");
            } catch (IOException e) {
                e.printStackTrace();
                Log.e("IO", "" + e);
            }
        } catch (JSONException e) {
            e.printStackTrace();
            Log.e("JSON", "" + e);
        }

        return json;
    }

    @Override
    protected void onPostExecute(JSONObject json) {
        try {
            if (json.getBoolean("success")) {
                // everything is ok
                SharedPreferences.Editor editor = mPreferences.edit();
                // save the returned auth_token into
                // the SharedPreferences
                editor.putString("AuthToken", json.getJSONObject("data").getString("auth_token"));
                editor.commit();

                // launch the HomeActivity and close this one
                Intent intent = new Intent(getApplicationContext(), HomeActivity.class);
                startActivity(intent);
                finish();
            }
            Toast.makeText(context, json.getString("info"), Toast.LENGTH_LONG).show();
        } catch (Exception e) {
            // something went wrong: show a Toast
            // with the exception message
            Toast.makeText(context, e.getMessage(), Toast.LENGTH_LONG).show();
        } finally {
            super.onPostExecute(json);
        }
    }
}
{% endhighlight %}

The code should be easy to follow. The important things are that we have to setup the params for our HTTP POST request, adding the user's email and password and setting the correct headers.
The method then tries to contact the request's url (our API) and if everything goes ok, return to the <code>onPostExecute()</code> method the results.

If the user exists and the password is correct, the <code>auth_token</code> will be returned from the API with the success info. We then proceed to save it in the <code>SharedPreferences</code> and launch the HomeActivity again, closing this one and showing a simple Toast.

Before switching to the <code>RegisterActivity</code> class, don't forget to add this new activity to the application's manifest.

{% highlight xml %}
<!-- file: AndroidManifest.xml -->
<activity
    android:name=".LoginActivity"
    android:label="@string/title_activity_login"
    android:noHistory="true" >
</activity>
{% endhighlight %}

The <code>android:noHistory</code> attribute is added for the same reasons we did it with the WelcomeActivity.

## RegisterActivity
Our app is almost completed (for the sake of this tutorial!) and what's left is adding the possibility for our potential users to register themselves directly on the app itself.

Create now (if you didn't do it before) a new Activity with the usual method and call it <code>RegisterActivity</code>

The layout file is again a ScrollView, containing our EditText fields for the user's information: email, username, password and password confirmation.

{% highlight xml %}
<!-- file: res/layout/activity_register.xml -->
<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="15dp" >

        <EditText
            android:id="@+id/userEmail"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:ems="10"
            android:hint="@string/user_email"
            android:inputType="textEmailAddress" >

            <requestFocus android:layout_width="wrap_content" />
        </EditText>

        <EditText
            android:id="@+id/userName"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:ems="10"
            android:hint="@string/user_name"
            android:inputType="text" />

        <EditText
            android:id="@+id/userPassword"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:ems="10"
            android:hint="@string/user_password"
            android:inputType="textPassword" />

        <EditText
            android:id="@+id/userPasswordConfirmation"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:ems="10"
            android:hint="@string/user_password_confirmation"
            android:inputType="textPassword" />

        <TextView
            android:id="@+id/registerDisclaimer"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:padding="10dp"
            android:text="@string/register_disclaimer"
            android:textAppearance="?android:attr/textAppearanceSmall" />

        <Button
            android:id="@+id/registerButton"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:onClick="registerNewAccount"
            android:text="@string/register" />
    </LinearLayout>

</ScrollView>
{% endhighlight %}

The implementation of the <code>RegisterActivity</code> class is really similar to what we did for the login.

The <code>registerNewAccount()</code> method is called whenever the user click on the register button. We then collect the data from the EditText fields and check if they are empty or not and also if the password and its confirmation are the same.

We then proceed to send all the information about the new user to the API, using again a custom implementation of the <code>
UrlJsonAsyncTask</code> class.

{% highlight java %}
// file: RegisterActivity.java
private final static String REGISTER_API_ENDPOINT_URL = "http://10.0.2.2:3000/api/v1/registrations";
private SharedPreferences mPreferences;
private String mUserEmail;
private String mUserName;
private String mUserPassword;
private String mUserPasswordConfirmation;

@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_register);

    mPreferences = getSharedPreferences("CurrentUser", MODE_PRIVATE);
}

public void registerNewAccount(View button) {
    EditText userEmailField = (EditText) findViewById(R.id.userEmail);
    mUserEmail = userEmailField.getText().toString();
    EditText userNameField = (EditText) findViewById(R.id.userName);
    mUserName = userNameField.getText().toString();
    EditText userPasswordField = (EditText) findViewById(R.id.userPassword);
    mUserPassword = userPasswordField.getText().toString();
    EditText userPasswordConfirmationField = (EditText) findViewById(R.id.userPasswordConfirmation);
    mUserPasswordConfirmation = userPasswordConfirmationField.getText().toString();

    if (mUserEmail.length() == 0 || mUserName.length() == 0 || mUserPassword.length() == 0 || mUserPasswordConfirmation.length() == 0) {
        // input fields are empty
        Toast.makeText(this, "Please complete all the fields",
            Toast.LENGTH_LONG).show();
        return;
    } else {
        if (!mUserPassword.equals(mUserPasswordConfirmation)) {
            // password doesn't match confirmation
            Toast.makeText(this, "Your password doesn't match confirmation, check again",
                Toast.LENGTH_LONG).show();
            return;
        } else {
            // everything is ok!
            RegisterTask registerTask = new RegisterTask(RegisterActivity.this);
            registerTask.setMessageLoading("Registering new account...");
            registerTask.execute(REGISTER_API_ENDPOINT_URL);
        }
    }
}
{% endhighlight %}

We need now to add the <code>RegisterTask</code> class implementation just like we did for the login. The details are really similar and maybe we can consider a refactor of the code to avoid some un-necessary receptions. But I leave this chore to you.

{% highlight java %}
// file: RegisterActivity.java
private class RegisterTask extends UrlJsonAsyncTask {
    public RegisterTask(Context context) {
       super(context);
    }

    @Override
    protected JSONObject doInBackground(String... urls) {
        DefaultHttpClient client = new DefaultHttpClient();
        HttpPost post = new HttpPost(urls[0]);
        JSONObject holder = new JSONObject();
        JSONObject userObj = new JSONObject();
        String response = null;
        JSONObject json = new JSONObject();

        try {
            try {
                // setup the returned values in case
                // something goes wrong
                json.put("success", false);
                json.put("info", "Something went wrong. Retry!");

                // add the users's info to the post params
                userObj.put("email", mUserEmail);
                userObj.put("name", mUserName);
                userObj.put("password", mUserPassword);
                userObj.put("password_confirmation", mUserPasswordConfirmation);
                holder.put("user", userObj);
                StringEntity se = new StringEntity(holder.toString());
                post.setEntity(se);

                // setup the request headers
                post.setHeader("Accept", "application/json");
                post.setHeader("Content-Type", "application/json");

                ResponseHandler<String> responseHandler = new BasicResponseHandler();
                response = client.execute(post, responseHandler);
                json = new JSONObject(response);

            } catch (HttpResponseException e) {
                e.printStackTrace();
                Log.e("ClientProtocol", "" + e);
            } catch (IOException e) {
                e.printStackTrace();
                Log.e("IO", "" + e);
            }
        } catch (JSONException e) {
            e.printStackTrace();
            Log.e("JSON", "" + e);
        }

        return json;
    }

    @Override
    protected void onPostExecute(JSONObject json) {
        try {
            if (json.getBoolean("success")) {
                // everything is ok
                SharedPreferences.Editor editor = mPreferences.edit();
                // save the returned auth_token into
                // the SharedPreferences
                editor.putString("AuthToken", json.getJSONObject("data").getString("auth_token"));
                editor.commit();

                // launch the HomeActivity and close this one
                Intent intent = new Intent(getApplicationContext(), HomeActivity.class);
                startActivity(intent);
                finish();
            }
            Toast.makeText(context, json.getString("info"), Toast.LENGTH_LONG).show();
        } catch (Exception e) {
            // something went wrong: show a Toast
            // with the exception message
            Toast.makeText(context, e.getMessage(), Toast.LENGTH_LONG).show();
        } finally {
            super.onPostExecute(json);
        }
    }
}
{% endhighlight %}

The code should be clear at this point and speak for itself.

Almost there, just remember to add the activity to the manifest and we are done!

{% highlight xml %}
<!-- file: AndroidManifest.xml -->
<activity
    android:name=".RegisterActivity"
    android:label="@string/title_activity_register"
    android:noHistory="true" >
</activity>
{% endhighlight %}

## Use the authentication
In order to use all the efforts we made until now, we need to edit just a line of code in our HomeActivity code. We have to pass the auth_token we set in the SharedPreferences after a successful login or registration and append it to the request url to the API.

{% highlight java %}
// file: HomeActivity.java
private void loadTasksFromAPI(String url) {
    GetTasksTask getTasksTask = new GetTasksTask(HomeActivity.this);
    getTasksTask.setMessageLoading("Loading tasks...");
    getTasksTask.execute(url + "?auth_token=" + mPreferences.getString("AuthToken", ""));
}
{% endhighlight %}

The Android app is now complete and it can be extended to be a fully capable application that consumes every data you want to provide via your API.

The very last thing to do is to uncomment the <code>before_filter :authenticate_user!</code> in the <code>TasksController</code> inside the Rails application. Without that filter every data is available without an authentication; with it an user can access to the data only with a proper authentication token.

## Bonus: ActionBarSherlock
Before concluding this long tutorial, I want to add just a little bonus that will enhance the user's experience of the app for the users that have not the latest and shiniest Android handset.

> ActionBarSherlock is an extension of the support library designed to facilitate the use of the action bar design pattern across all versions of Android with a single API.
>
> The library will automatically use the native action bar when appropriate or will automatically wrap a custom implementation around your layouts. This allows you to easily develop an application with an action bar for every version of Android from 2.x and up.
>
> Source: [actionbarsherlock.com](http://www.actionbarsherlock.com)

### Installing ActionBarSherlock
Download and extract the actionbarsherlock files from the site and add the library to the current workspace in Eclipse: <code>"File > New > Other ... > Android Project from Existing Code"</code>

Navigate the ActionBarSherlock directory and select the "library" dir. Ignore the warning/error that may arise and right-click on the new project and select <code>"Refactor > Rename"</code> and rename the project *"ABS 4.0"* (or whatever you like).

Right-click on the AuthExample project and select <code>"Properties"</code>: click on the <code>"Android"</code> section and add the just added ABS library in the <code>"Library"</code> sub-section.

Eclipse should now give you an error like this:

{% highlight bash %}
Found 2 versions of android-support-v4.jar in the dependency list, but not all the versions are identical (check is based on SHA-1 only at this time).
All versions of the libraries must be the same at this time.
Jar mismatch! Fix your dependencies
{% endhighlight %}

The problem can be solved by deleting the <code>android-support-v4.jar</code> in the <code>"libs"</code> directory of our projects: right-click on it, browsing the Package Explorer and select <code>"delete"</code>.

### Updating the existing code
To use all the nice features that ABS provides, we need to alter some code, starting from changing the parent class of our Activities. Instead of extending the <code>android.app.Activity</code>, we need to extend the <code>SherlockActivity</code> like so:

{% highlight java %}
// file: HomeActivity.java
import com.actionbarsherlock.app.SherlockActivity;

public class HomeActivity extends SherlockActivity {
// activity implementation
}
{% endhighlight %}

Do the same for all the other activities and remember to remove the old imports of <code>import android.app.Activity;</code>.

In the HomeActivity there will be some problems with the code that manage the creation of the Menu. The fix is quick and consists in changing the method called to get the menu inflater (<code>getMenuInflater()</code>) with  <code>getSupportMenuInflater()</code> provided by the SherlockActivity class.

{% highlight java %}
// file: HomeActivity.java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getSupportMenuInflater().inflate(R.menu.activity_home, menu);
    return true;
}
{% endhighlight %}

Finally, change the default theme for the application in the <code>AndroidManifest.xml</code> file.

{% highlight xml %}
<!-- file: AndroidManifest.xml -->
<application
        android:icon="@drawable/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/Theme.Sherlock" >
{% endhighlight %}

You can choose the default dark one or <code>Theme.Sherlock.Light</code> and <code>Theme.Sherlock.Light.DarkActionBar</code>. For more information, check out the official documentation at [actionbarsherlock.com/theming](http://actionbarsherlock.com/theming.html)

# Conclusion
That's all! It was a wild ride, spanning a lot of topics and languages. I hope you enjoyed and if you have anything to ask or critics to raise, let me know.

I hope the information provided with this tutorial could be the basis for wonderful applications. I tried to write it as the tutorial I wished I had had when I started addressing these topics in my apps coding.

I wish it could helpful to you now.

Bye,
Luca