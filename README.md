# Silverstripe Security Extensions

[![CI](https://github.com/silverstripe/silverstripe-security-extensions/actions/workflows/ci.yml/badge.svg)](https://github.com/silverstripe/silverstripe-security-extensions/actions/workflows/ci.yml)
[![Silverstripe supported module](https://img.shields.io/badge/silverstripe-supported-0071C4.svg)](https://www.silverstripe.org/software/addons/silverstripe-commercially-supported-module-list/)

## Overview

This module is a polyfill for some security related features that will become part of the core SilverStripe
product, but are required for older Silverstripe 3.7 and 4.x support in the meantime.

This module will _not_ be made compatible with CMS 5 - instead, its functionality has been folded back into the core modules.

## Installation

```
$ composer require silverstripe/security-extensions 1.x-dev
```

## Features

### Sudo mode

Sudo mode represents a heightened level of permission in that you are more certain that the current user is actually
the person whose account is logged in. This is performed by re-validating that the account's password is correct, and
will then last for a certain amount of time (configurable) until it will be checked again.

Sudo mode will automatically be enabled for the configured lifetime when a user logs into the CMS. Note that if the
PHP session lifetime expires before the sudo mode lifetime, that sudo mode will also be cleared (and re-enabled when
the user logs in again). If the user leaves their CMS open, or continues to use it, for an extended period of time
with automatic refreshing in the background, sudo mode will eventually deactivate once the max lifetime is reached.

#### Configuring the lifetime

The default `SudoModeServiceInterface` implementation is `SudoModeService`, and its lifetime can be configured with
YAML. You should read the lifetime value using `SudoModeServiceInterface::getLifetime()`.

```yaml
SilverStripe\SecurityExtensions\Service\SudoModeService:
  lifetime_minutes: 25
```

#### Enabling sudo mode for controllers

You can add the `SilverStripe\SecurityExtensions\Services\SudoModeServiceInterface` as a dependency to a controller
that requires sudo mode for one of its actions:

```php
class MyController extends Controller
{
    private $sudoModeService;

    private static $dependencies = ['SudoModeService' => '%$' . SudoModeServiceInterface::class];

    public function setSudoModeService(SudoModeServiceInterface $sudoModeService): self
    {
        $this->sudoModeService = $sudoModeService;
        return $this;
    }
}
```

Performing a sudo mode verification check in a controller action is simply using the service to validate the request:

```php
public function myAction(HTTPRequest $request): HTTPResponse
{
    if (!$this->sudoModeService->check($request->getSession()) {
        return $this->httpError(403, 'Sudo mode is required for this action');
    }
    // ... continue with sensitive operations
}
```

### Using sudo mode in a React component

This module defines a [React Higher-Order-Component](https://reactjs.org/docs/higher-order-components.html) which can
be applied to React components in your module or code to intercept component rendering and show a "sudo mode required"
information and log in screen, which will validate, activate sudo mode, and re-render the wrapped component afterwards
on success.

**Note:** the JavaScript injector [does not currently support injecting transformations/HOCs](https://github.com/silverstripe/react-injector/issues/4),
so we have coupled the application of these [injector transformations](https://docs.silverstripe.org/en/4/developer_guides/customising_the_admin_interface/reactjs_redux_and_graphql/#transforming-services-using-middleware)
into this module itself for the silverstripe/mfa module. Unfortunately, if you want to apply this to your own code
you will need to either duplicate the `SudoMode` HOC into your project or module and apply the transformation at that
point.

![Sudo mode HOC example](docs/_images/sudomode.png)

Example implementation:

```jsx
import WithSudoMode from '../containers/SudoMode/SudoMode';

Injector.transform('MyComponentWithSudoMode', (updater) => {
  updater.component('MyComponent', WithSudoMode);
});
```

#### Requirements for adding to a component

While the `sudoModeActive` prop is gathered automatically from the Redux configuration store, backend validation is
also implemented to ensure that the frontend UI cannot simply be tampered with to avoid re-validation on sensitive
operations.

Ensure you protected your endpoints from [cross site request forgery (CSRF)](https://docs.silverstripe.org/en/4/developer_guides/forms/form_security/#cross-site-request-forgery-csrf)
at the same time.

### Require password change on next log in

Administrators with the ability to administer members can see a checkbox in the CMS under the area to set the member's password.
Checking this box will set the password expiry to the current date, meaning the next time the member logs in they will be required to choose a new password for their account.

The date is set selectively in order to not batter the database with updates to that member's records each time an unrelated setting is changed and saved. The matrix is as follows (`-` indicates no change):

 Expiry Date  | Checked   | Unchecked
--------------|-----------|-----------
 **Null**     | now       | -
 **Future**   | now       | -
 **Expired**  | -         | null

No change is made when setting this field and the password is already expired for auditing purposes (an administrator could see how long ago a password expired).

Similarly no change is made when unsetting this field and the expiry date is in the future, it should remain so - the checkbox is for immediately requiring a new password on the _next_ log in.

Given the above two paragraphs, it should not be possible to reach these cases under normal (CMS) usage, as the UI reflects the current state of the PasswordExpiry field on load. The checkbox will be checked if the current password is already expired.

## Versioning

This library follows [Semver](http://semver.org). According to Semver,
you will be able to upgrade to any minor or patch version of this library
without any breaking changes to the public API. Semver also requires that
we clearly define the public API for this library.

All methods, with `public` visibility, are part of the public API. All
other methods are not part of the public API. Where possible, we'll try
to keep `protected` methods backwards-compatible in minor/patch versions,
but if you're overriding methods then please test your work before upgrading.

## Reporting Issues

Please [create an issue](https://github.com/creative-commoners/silverstripe-security-extensions/issues)
for any bugs you've found, or features you're missing.

## License

This module is released under the [BSD 3-Clause License](LICENSE.md).
