# OuiHop SDK

## Configure SDK Environment Parameters
To configure SDK with environment parameters, first environment variables should be set for `OuiHopSDKConfiguration` static class:

- OuiHop backend base URL.
- Google APIs key.
- Firestore configuration plist file path. If nothing is provided, default Firestore configuration plist file path is assumed.
- Client token. If nothing was provided, all API calls will return errors, as backend won’t be able to identify calling party.

After those values are properly set, configure method should be called. This should be done before any other interactor is used.

## Sign up
To sign up a new user, `UserInteractor` singleton class is used. There are two types of users:

- OuiHop: This is the same type of users that exists in OuiHop application. This kind of users is meant to last for long. Those users can be used in OuiHop application directly. OuiHop users can be identified by either email/password or facebook id/access token pairs.
  - Used method: `signupOuiHopUser:successHandler:errorHandler:`
  - Required fields: `email`, `password`, `mobileNo`, `country`, `gender`, `birthdate`, `firstName`, `lastName`.
- Third Party: This type of users is used to register temporary users to fulfil a trip request or something in a third party application. This is the main type to be used along with the SDK.
  - Used method: `signupThirdPartyUser:successHandler:errorHandler:`
  - Required fields: `mobileNo`, `password`.

## Authentication
To authenticate, `UserInteractor` singleton class is used. Three methods are available for authentication:

- OuiHop (email/password): `loginWithOuiHopEmail:password:`
- OuiHop (facebook id/token): `loginWithExternalId:accessToken:`
- Third party: `loginWithMobileNumber:password:`

All of the above methods fills UserInteractor with a OuiHop access token. This logged in user is cached, so next time SDK is initialized, the previous token is retrieved. Once logged out, caches are cleared.

## User profile
After authentication, user full profile should be retrieved using `UserInteractor` method `getUserProfile`.

For users already cached, just call silentLogin method to retrieve user profile. This method could be used any time in the application as well to refresh profile values.

## User mode
Users could be either a pedestrian or driver. During signup, if a user includes a car, it is assumed to be a driver. If not, it will be a pedestrian.

User mode can be changed through `UserInteractor` using `changeUserMode:`. User mode is stored on server and doesn’t change even if application was terminated.

User mode affects functions that can be called through the SDK. For example, a trip cannot be created while currently authenticated user is in pedestrian mode.

## User location 
SDK uses a `LocationInteractor` singleton class to monitor user location. At any stage, latest location can be retrieved from this class, along with delegate methods to monitor change. Another class `SendLocationInteractor` is automatically used to keep backend posted with user location. Frequency of updating backend with user location depends on user mode and status. This is maintained automatically as well. There’s no need to use `SendLocationInteractor` in a direct manner, as `LocationInteractor` class is sufficient for most cases.

All what is needed is to call `startMonitoringLocation` at application start. `LocationInteractor` will take care of gathering location information, and synchronizing it with backend.

> **Important:** In order for `LocationInteractor` to work properly, you must do the following:
> 
>- Make sure “Location” is included in `UIBackgroundModes` in your application `Info.plist`.
>- Require “Always” location permission before calling `startMonitoringLocation` method.

## Surrounding users
To retrieve surrounding users, `LocationInteractor` should be active first. OuiHop SDK uses near real-time algorithms to notify your application when new users appear, or old ones update. This can be done through `SurroundingUsersInteractor`. All you have to do is to specify an accuracy for the search, and an interval to search for nearby users.

Accuracy specifies the length of the GeoHash being used to search. You can read more about the GeoHash mechanism at [Wikipedia](https://en.wikipedia.org/wiki/Geohash).

After initializing the interactor, you need to register as an observer and wait for update events.

## Surrounding trips (pedestrian)
`SurroundingTripsInteractor` is meant for pedestrians to detect trips around. However, there are some dependencies that assist this interactor in retrieving interesting trips:

`LocationInteractor` must be started and monitoring location.
During `SurroundingTripsInteractor` initialization, it must be feeded with a `SurroundingUsersInteractor` that is already initialized. This helps excluding trips whose drivers are very far, or already passed pedestrian locaiton.
During `SurroundingTripsInteractor` initialization, it could be feeded with a `PedestrianDestinationInteractor`. This helps retrieving trips that match pedestrian destination only.

Once initialized, you can register as an observer for `SurroundingTripsInteractor` and receive trips updates.

## Starting a trip (driver)
Using `DriverTripInteractor`, a driver can create a new trip. The interactor is straightforward and provides needed information during the trip, and after termination.

## Managing requests (driver)
As an auxiliary to `DriverTripInteractor`, you can manage requests from pedestrians through `DriverTripRequestsInteractor`. This helps accepting, rejecting, and proceeding with requests. One instance of the interactor manages all requests of the current trip. A `DriverTripRequestsInteractor` should be initialized with a `DriverTripInteractor` that has an active trip.

When a request isn’t boarded yet, it’s considered an **active** request. Once it’s boarded, it is considered a **boarded** request. If a request is cancelled at any moment, or arrived, it’s considered a **finished** request.

For active and boarded requests, pedestrians’ location is tracked in real time, to provide the most accurate experience.

## Requesting to join a trip (pedestrian)
For each request from a pedestrian to a driver, a `PedestrianTripRequestInteractor` instance should be created. Actions can later take place through each corresponding interactor. Changes from driver side are passed through an observer pattern to stay updated with request state changes in real-time.

## Pedestrian destination creation (pedestrian)
For a pedestrian to publish their destination in order for drivers to propose rides, PedestrianDestinationInteractor should be used. After publishing destination, the new `PedestrianDestinationInteractor` instance should be passed to `SurroundingTripsInteractor` in order to filter new trips targeting only trips with suitable destination.

## Fulfilling a pedestrian destination (driver)
`PedestrianDestinationRequestInteractor` is responsible for listening to all created pedestrian destinations. The purpose is to use that pedestrian destination to propose a new trip to fulfil this pedestrian destination.
