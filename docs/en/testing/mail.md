# Testing — Mailer Tests

Spiral has a convenient way to test the [spiral/sendit](../component/sendit.md) component. One useful
feature is the ability to use a fake mailer to prevent actual emails from being sent. This is useful because sending
emails is typically not related to the code you are testing.

The `fakeMailer()` method replaces the `Spiral\Mailer\MailerInterface` with a fake mailer
`Spiral\Testing\Mailer\FakeMailer` in the container. It provides useful assertion methods to check if an email was sent,
how many times, and what the email contains.

Here's an example of how you might use this in a test case for a user registration process:

```php
use Spiral\Mailer\Message;
use Tests\TestCase;

final class UserServiceTest extends TestCase
{
    private \Spiral\Testing\Mailer\FakeMailer $mailer;

    protected function setUp(): void
    {
        parent::setUp();
        $this->mailer = $this->fakeMailer();
    }

    public function testRegisterUser(): void
    {
        // Perform user registration ...

        $this->mailer->assertSent(WelсomeMessage::class);
    }
}
```

The `assertSent()` method provides a way to check if a specific mailable was sent and to check if it contains certain
information. You can pass a closure to it to specify the conditions that a mailable must meet in order for the
assertion to be successful.

```php
use Spiral\Mailer\Message;

$this->mailer->assertSent(
    WelcomeMessage::class,
    static fn (Message $message) => \in_array('user@site.com', $message->getTo())
);
```

For example, it is being used to check if a mailable of class `WelcomeMessage` was sent, and it passes a closure that
checks if the recipient email address is `user@site.com`. If at least one mailable of class `WelcomeMessage` was sent
with a recipient email address, the assertion will be successful.

In addition, there is assertSentTimes() method. It can be used to assert that a specific mailable was sent a certain
number of times. It takes two arguments: the class name of the mailable and the expected number of times it should have
been sent.

```php
$this->mailer->assertSentTimes(WelcomeMessage::class, 2);
```

Yes, in addition, the `FakeMailer` provides `assertNotSent()` and `assertNothingSent()` methods. These methods can be
used to assert that a specific mailable was not sent or that no mail was sent at all, respectively.

`assertNotSent()` method is used to assert that a specific mailable was not sent. It takes the same closure
as `assertSent()` method, but it asserts that the closure should fail for all the sent mailables.

```php
$this->mailer->assertNotSent(
    WelcomeMessage::class,
    static fn (Message $message) => \in_array('user@site.com', $message->getTo())
);
```

`assertNothingSent()` method, on the other hand, asserts that no mail was sent during the test case. This is useful if
you want to make sure that no unwanted mail is sent during your test.

```php
$this->mailer->assertNothingSent();
```