---
title: "Smooth sign-ups for product led growth"
datePublished: Fri Mar 08 2024 13:11:11 GMT+0000 (Coordinated Universal Time)
cuid: cltiodmgm000f09l224jk18m8
slug: smooth-sign-ups-for-product-led-growth
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/2iXKHA9PjVk/upload/e74ecb77640fab5273042bba19c70ee4.jpeg

---

## Background

As an Indie Hacker, I’m building an influencer marketing marketplace that matches appropriate influencers with the brands for marketing campaigns. Since this is a new platform, the focus is primarily on onboarding users. Mainly influencers at this point. With this product, influencers can create personalized and dedicated webpage where they can showcase previous influencer marketing campaigns and accept orders.

This blog post dives into the journey of building smooth sign-up flows to onboard influencers with the least amount of friction.

## What is Product Led Growth

Unlike traditional sales-driven approaches, where sales teams actively push products onto customers, PLG relies on the inherent value and usability of the product to drive adoption and conversion. The product is designed for users to easily sign up, set up, and start using it without requiring assistance. A good example of this would be Discord, Notion.

In this post, we’ll be looking at the PLG from a product engineer's perspective and understand the kind of product and engineering challenges that need to be solved to create a smooth sign-up experience.

## The Sign Up Experience

Sign-ups are often a source of friction causing drop-offs, as users are bombarded with forms that ask for personal information and giving impeccable importance to user authentication over allowing users to experience the value that the product offers.

Two main problems need to be solved

1. Authenticating user only when necessary
    
2. Asking for personal information in a minimally intrusive way
    

## Onboarding experience

The fundamental value the product delivers is the webpage where content creators can showcase their work related to influencer marketing campaigns for various brands and accept orders.

Now the information I need from the user to deliver a minimum viable landing page is personal information like Name, Username(for creating a unique URL), About, profile image, Instagram URL, and YouTube channel URL.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1709902587260/b3a67e00-27ab-4b36-a4e3-ff10b8d790f8.gif align="center")

* The Get Started button on the landing page will directly take the user to the dashboard
    
* To setup the landing page, it asks for personal information and immediately shows the preview.
    
* On clicking the publish button it then asks email for authenticating user.
    
* The influencer landing page is available at the URL built from the selected username.
    

## The framework

Typical signup flow —

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1709902816588/11bca262-d759-4079-a158-4ecd7c7ae05d.png align="center")

Revised signup flow —

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1709902843779/292d4757-bb08-4cb3-8c8a-a21ac677c405.png align="center")

## The principles

### Inverting the order

Typically, the product onboarding flow follows the approach of first authenticating the user, then requesting the required personal information, and finally navigating the user to the product. The problem with this approach is that the value the product offers is placed at the end of the journey, which may cause users to drop off during the first and second steps.

By inverting the order, the user first sees the product offering. If they find it useful and relevant, then they proceed to signing up.

### Combining information collection with value delivery

In the revised flow, you can see that personal information and the product are shown together. This provides a less intrusive experience because while providing information, the user is also experiencing the value delivered by the product.

## Engineering

Engineering of this product flow involves maintaining verified vs unverified accounts, addressing security concerns, restricting access, and a purging mechanism for unverified accounts.

### Verified vs Unverified accounts

There’s a flag in the user table indicating the verification status of each account. Upon completing the initial onboarding process which includes providing personal information and setting up the front page, the account is marked as unverified. After the user completes the email authentication step, the account status is updated to verified status.

This verification status can be utilized throughout the application to control access to certain features or resources. For instance, unverified accounts would have access to limited functionality until they are verified.

### Authentication middleware

The primary role of middleware is to determine whether a user is verified or unverified before granting access to the application.

If the user is unverified, the middleware should first check if the user has set up a landing page. If the landing page is already set up, the middleware should prompt the user to verify their email address. If the landing page has not been set up, the middleware should allow the user to do so.

If the user is verified, the middleware should validate the JWT session cookie.

Following is the code for NextJS middleware

```jsx
export async function middleware(request: NextRequest) {
  const session = request.cookies.get("session")?.value;
  const payload = await decrypt(session);

  // Some more application specific checks and coditions here...

  if (!payload) {
    const response = await selectLandingPageApi(userId);

    // Allow user to access dashboard if frontpage is not created
    // This is done for the PLG and smooth sign up experience.
    if (response.status === 404) {
      return;
    }

    return redirectToLogin(request);
  }

  // Redirect to login page if malicious user is trying to access
  // data of other user.
  if (payload.user.userId !== userId) {
    return redirectToLogin(request);
  }

  // Access granted
  return;
}

export const config = {
  matcher: ["/dashboard/:path*"],
};

```

### Periodic purging of unverified accounts

To manage unverified accounts and prevent the accumulation of inactive accounts, there’s a periodic deletion job that runs weekly to identify and delete unverified accounts that have remained unverified for more than 7 days.

The deletion cron job queries the user's table for accounts with unverified status and checks their creation date. Accounts that have not been verified within the timeframe are considered inactive and eligible for deletion.

By regularly purging unverified accounts, we can maintain a clean and active user database, and improve system performance.

## Conclusion

In conclusion, implementing a product-led growth strategy for user sign-ups can greatly enhance the user onboarding experience. By inverting the order and allowing users to experience the product's value before asking them to sign up, we can reduce drop-offs and increase user engagement. Furthermore, combining information collection with value delivery presents a less intrusive way of gathering necessary data while maintaining user interest.

The technical implementation of this strategy involves carefully managing verified and unverified accounts, employing middleware for access control, and implementing a mechanism for the periodic purging of unverified accounts.