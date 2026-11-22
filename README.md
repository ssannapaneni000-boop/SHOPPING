# FreshCart — Flutter Grocery App Starter

A green-and-white Flutter starter for a grocery shopping app. It includes customer navigation, sample products, cart state, Firebase-ready authentication/data services, and a basic admin product screen.

## Important
This is a runnable **starter project**, not a finished production commerce platform. You must connect your own Firebase project and configure security rules before using real customer data. Razorpay is not wired to a live merchant account; production payments require a trusted backend to create orders and verify signatures. Never put a Razorpay secret key in Flutter.

## 1. Install tools (Windows)
1. Install Flutter SDK and add `flutter\bin` to PATH.
2. Install Android Studio (Android SDK) or enable Windows Desktop in Visual Studio with **Desktop development with C++**.
3. Install Node.js only if you need Firebase CLI.
4. Open this folder in VS Code or Android Studio.

Check:
```bash
flutter doctor
```

## 2. Run in demo mode
From this folder (same folder as `pubspec.yaml`):
```bash
flutter pub get
flutter run -d chrome   # Web
flutter run -d windows  # Windows desktop
```
Demo mode uses sample products and demo sign-in. It does not store real accounts.
Android, iOS, Linux, and macOS platform projects are also included. Build each
target on a machine with that platform's required development tools installed.

## 3. Connect Firebase
1. Create a project in Firebase Console.
2. In Authentication → Sign-in method, enable **Email/Password**.
3. Create a Cloud Firestore database and (if using product images) enable Storage.
4. Install/configure FlutterFire CLI:
```bash
dart pub global activate flutterfire_cli
firebase login
flutterfire configure
```
Choose your Firebase project and the platforms you need. This generates `lib/firebase_options.dart`.
5. In `lib/main.dart`, uncomment the Firebase initialization block shown in the comments and import `firebase_options.dart`; set `useFirebase = true` in `lib/services/app_config.dart`.
6. Run:
```bash
flutter pub get
flutter analyze
flutter run -d windows
```
Firebase plugins on Windows may have platform limitations; Android/iOS are the usual production targets. Test on the target platform before deployment.

## 4. Firestore structure (suggested)
- `products/{productId}`: `name`, `description`, `category`, `price`, `unit`, `emoji`, `imageUrl`, `stock`, `isActive`
- `orders/{orderId}`: `userId`, `items`, `subtotal`, `deliveryFee`, `total`, `paymentMethod`, `status`, `createdAt`
- `users/{uid}`: `name`, `email`, `role` (`customer` or `admin`)
- `lists/{listId}`: `userId`, `title`, `items`

Do not let clients choose their own admin role. Assign admin roles securely from a trusted environment and write restrictive Firestore rules before deployment.

## The three panels
| Panel | Who | How to enter | What they can do |
|---|---|---|---|
| **Guest** | anyone, no account | Login screen → **Continue as guest** | browse products & categories, search, view details, use the cart. Checkout asks them to login/sign up (cart is kept). |
| **User** | registered customer | Login / Create account | everything a guest can + checkout, order history, favorites, grocery lists, profile |
| **Admin** | `role == "admin"` | home ⋮ menu → **Admin panel** (or separate admin app) | dashboard, orders, products, categories, users, inventory, reports, settings … |

Guests never see the Admin option, and placing an order is blocked for guests in code as well.

## Admin panel
The admin panel code is in `lib/admin/`. It can be opened two ways:
1. **Inside the customer app**: tap the ⋮ menu on the home screen → **Admin panel** (shown only to admin accounts; in demo mode everyone sees it).
2. **As a separate app** (entry point `lib/main_admin.dart`):
```bash
flutter run -t lib/main.dart         # customer app
flutter run -t lib/main_admin.dart   # admin app
```
It includes:
- **Dashboard**: today's sales, pending orders, product count, low-stock alert
- **Orders**: filter by status, view items/address, update status (Placed → Packed → Out for delivery → Delivered / Cancelled)
- **Products**: search, add, edit (price, stock, category, emoji), remove
- **Categories**: add / remove (blocked while products still use it)
- **Users**: list with roles (read-only)

Demo mode: any email/password signs in as admin, but the two apps run separately, so demo orders from the customer app do NOT appear in the admin app. Connect Firebase to share data between them.
Firebase mode: admin = `users/{uid}.role == "admin"`. Set it manually in the Firebase console on your own user document.
More admin features: inventory/restock, sales reports, bulk pricing, coupons, banners, delivery slots, delivery partners (assign to orders), announcements, activity log, store settings (delivery fee, minimum order, open/closed, low-stock level), order search, CSV export, order notes, product duplicate, customer insights.
Extra Firestore collections: `coupons`, `banners`, `delivery_slots`, `delivery_partners`, `announcements`, `activity_log`, `settings/store`. Add `allow read, write: if isAdmin();` rules for them (and `allow read: if true;` for `settings`).
Extra Firestore collection: `categories/{name}`: `name`.

Example Firestore rules to start with (review before launch):
```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    function isAdmin() { return request.auth != null && get(/databases/$(db)/documents/users/$(request.auth.uid)).data.role == 'admin'; }
    match /products/{id}   { allow read: if true; allow write: if isAdmin(); }
    match /categories/{id} { allow read: if true; allow write: if isAdmin(); }
    match /users/{uid} {
      allow read: if request.auth.uid == uid || isAdmin();
      allow create: if request.auth.uid == uid && request.resource.data.role == 'customer';
      allow update: if isAdmin();
    }
    match /orders/{id} {
      allow create: if request.auth != null && request.resource.data.userId == request.auth.uid && request.resource.data.status == 'Placed';
      allow read: if isAdmin() || resource.data.userId == request.auth.uid;
      allow update: if isAdmin();
    }
  }
}
```
Note: customer screens still read the local product list; switch them to `ProductService().getProducts()` when you go live with Firebase.

## 5. Payment integration
Cash on Delivery is represented in the checkout demo. For Razorpay, create a backend endpoint (Cloud Functions or your server) that creates a Razorpay order and verifies the payment signature. Do not embed Razorpay secret credentials in this app.

## Troubleshooting
- `firebase_options.dart` missing: run `flutterfire configure` from this project folder.
- Firebase `app/no-app`: ensure initialization happens before `runApp`.
- Login provider disabled: enable Email/Password in Firebase Authentication.
- `flutter analyze` errors: run `flutter pub get`, then copy the full error text for diagnosis.
