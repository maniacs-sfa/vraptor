---
title: Validation
layout: page
section: 7
categories: [en, docs]
---

VRaptor3 supports two different validation styles: classic and fluent. The starting point to both styles is the Validator interface. In order to access the Validator, your resource must receive it in the constructor:

{% highlight java %}
import br.com.caelum.vraptor.Validator;
...

@Resource
class EmployeeController {
    private Validator validator;

    public EmployeeController(Validator validator) {
        this.validator = validator;
    }
}
{% endhighlight %}

<h3>Classic style</h3>

The classic style is very similar to VRaptor2's validation. Inside your business logic, all you have to do is check the data you want, and if you find any validation errors, add them to the errors list. For example, to validate that employee name is 'John Doe':

{% highlight java %}
public void add(Employee employee) {
    if (!employee.getName().equals("John Doe")) {
        validator.add(new ValidationMessage("invalid.name", "error"));
    }

    validator.onErrorUsePageOf(EmployeeController.class).form();

    dao.add(employee);
}
{% endhighlight %}

When you call validator.onErrorUse, if there are any validation errors, VRaptor will stop execution and redirect to the page you specified. This redirect has the same behavior as the result.use(..) redirects.

<h3>Fluent style</h3>

The goal of fluent style is to write the validation code in such way that it feels natural. For example, if we want the employee name to be required:

{% highlight java %}
public add(Employee employee) {
    validator.checking(new Validations() { {
        that(!employee.getName().isEmpty(), "error", "name.is.required");
    } });

    validator.onErrorUsePageOf(EmployeeController.class).form();

    dao.add(employee);
}
{% endhighlight %}

You can read the code above like this: "Validator, check my validations. First one is that employee name cannot be empty". Much closer to natural language.
So, if employee name is empty, the flow will be redirected to the "form" logic, which shows the user a form to insert employee data again. Also, the error message is sent back to the form.
There are validations that may occur only if other validation succeeded, for instance I will check user age only if the user is not null. The that method will return a boolean that represents the success of the validation:

{% highlight java %}
validator.checking(new Validations() { {
    if (that(user != null, "user", "null.user")) {
        that(user.getAge() >= 18, "user.age", "user.is.underage");
    }
} });
{% endhighlight %}

So the second validation will execute only if the first didn't fail.
Validation with message parameters
You can add parameters to you message keys:

{% highlight jsp %}
greater.than = {0} should be greater than {1}
{% endhighlight %}

And use it on your validations code:

{% highlight java %}
validator.checking(new Validations() { {
    that(user.getAge() >= 18, "user.age", "greater.than", "Age", 18);
    // Age should be greater than 18
} });
{% endhighlight %}

You can even i18n your parameters, with i18n method:

{% highlight jsp%}
user.age = User age
{% endhighlight %}

{% highlight java %}
validator.checking(new Validations() { {
    that(user.getAge() >= 18, "user.age", "greater.than", i18n("user.age"), 18);
    // User age should be greater than 18
} });
{% endhighlight %}

<h3>Validation using Hamcrest Matchers</h3>

You can use Hamcrest matchers for making validation even more fluent and readable, with the advantage of matcher composition and the creation of new matchers that Hamcrest allows:

{% highlight java %}
public admin(Employee employee) {
    validator.checking(new Validations() { {
        that(employee.getRoles(), hasItem("ADMIN"), "admin", "employee.is.not.admin");
    } });

    validator.onErrorUsePageOf(LoginController.class).login();

    dao.add(employee);
}
{% endhighlight %}

<h3>Bean Validation</h3>

VRaptor 3 also supports Bean Validation. To use these features you only need to put any implementation of Bean Validation in your classpath.
In the example above, to validate the employee object, just add one line to your code:

{% highlight java %}
public add(Employee employee) {
    // Validation with Bean Validation or Hibernate Validator
    validator.validate(funcionario);

    validator.checking(new Validations() { {
        that(!employee.getName().isEmpty(), "error", "name.is.required");
    } });

    validator.onErrorUsePageOf(EmployeeController.class).form();

    dao.add(employee);
}
{% endhighlight %}

Since 3.5 release, there is support to method validation. You need only to put any implementation of Bean Validator 1.1 in the classpath. See example bellow:

{% highlight java %}
public adiciona(@NotNull String name, @Future dueDate) {
    ...
}
{% endhighlight %}

<h3>Where to redirect in case of errors</h3>

Another issue that one must consider when validating data is where to redirect when an error occurs. How do one redirect the user to another resource using VRaptor3 in case of validation errors?
Easy, just tell your validator to do just that: when you find any validation error, send the user to the specified resource. See the example:

{% highlight java %}
public add(Employee employee) {

    // Fluent validation
    validator.checking(new Validations() { {
        that(!employee.getName().isEmpty(), "error", "name.is.required");
    } });

    // Classic validation
    if (!employee.getName().equals("John Doe")) {
        validator.add(new ValidationMessage("error", "invalid.name"));
    }

    validator.onErrorUse(page()).of(EmployeeController.class).form();

    dao.add(employee);
}
{% endhighlight %}

If your logic may add any validation error you <strong>must</strong> specify where to go in case of error. Validator.onErrorUse works just like result.use: you can use any view from Results class.
Shortcuts on Validator
Some redirections are pretty common, so there are shortcuts on Result interface for them. The available shortcuts are:

<ul>
	<li>validator.onErrorForwardTo(ClientController.class).list() ==> validator.onErrorUse(logic()).forwardTo(ClientController.class).list();</li>

	<li>validator.onErrorRedirectTo(ClientController.class).list() ==> validator.onErrorUse(logic()).redirectTo(ClientController.class).list();</li>

	<li>validator.onErrorUsePageOf(ClientController.class).list() ==> validator.onErrorUse(page()).of(ClientController.class).list();</li>

	<li>validator.onErrorSendBadRequest()	 ==> validator.onErrorUse(status()).badRequest(errors);</li>
</ul>

Furthermore, if one are redirecting to a method on the same controller, one can use:

<ul>
	<li>validator.onErrorForwardTo(this).list() ==> validator.onErrorUse(logic()).forwardTo(this.getClass()).list();</li>

	<li>validator.onErrorRedirectTo(this).list() ==> validator.onErrorUse(logic()).redirectTo(this.getClass()).list();</li>

	<li>validator.onErrorUsePageOf(this).list() ==> validator.onErrorUse(page()).of(this.getClass()).list();</li>
</ul>

<h3>L10N with the bundle of the Localization</h3>

To force translation the parameters of the i18n() methods, simply inject the Localization and use it bundle on the constructor of the Validations():

{% highlight java %}
private final Validator validator;
private final Localization localization;

public UserController(Validator validator, Localization localization) {
    this.validator = validator;
    this.localization = localization;
}
validator.checking(new Validations(localization.getBundle()) { {
    that(user.getAge() >= 18, "user.age", "greater.than", i18n("user.age"), 18);
} });
{% endhighlight %}

Note that here is passed manually the ResourceBundle, so the key "user.age" will be translated correctly in the change of the Locale.
Showing validation errors on JSP
When there are validation errors, VRaptor will set a request attribute named errors with the error list, so you can show them on your JSP with:

{% highlight jsp %}
<c:forEach var="error" items="${errors}">
    ${error.category} - ${error.message}<br />
</c:forEach>
{% endhighlight %}