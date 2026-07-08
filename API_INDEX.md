# Hawdaj — API Index

> كل endpoints الموبايل تحت `/api` · القاعدة: prod `https://dashboard.hawdaj.net/api/` · test `https://test.dashboard.hawdaj.net/api/`.
> Auth = Sanctum Bearer. ✅ = الميزة محلّلة بالكامل (راجع ملفها). المسارات مجمّعة حسب الميزة.
> ملاحظة: مسارات المستخدم تعتمد فحص `auth('api')` داخل الـ controller (غالبًا بلا middleware صريح على الراوت).

---

## 01 — Authentication & Profile ✅ (`FEATURES/01_Authentication.md`)
| Method | Endpoint | Flutter (feature) | Laravel Controller |
|---|---|---|---|
| POST | `register` | auth · RegisterCubit | `Api/Auth/AuthenticationController@register` |
| POST | `login` | auth · LoginCubit | `AuthenticationController@login` |
| POST | `logout` | auth · LogoutCubit | `AuthenticationController@logout` |
| POST | `social/callback` | auth · SocialAuthCubit | `Api/Auth/SocialController@callback` |
| POST | `forgot-password` | auth · ForgotPasswordCubit | `Api/Auth/ForgotPasswordController@forgot` |
| POST | `verify-email-code` | auth · (VerifyEmailCodeCubit stub) | `ForgotPasswordController@verify` |
| POST | `reset-password` | auth | `Api/Auth/ResetPasswordController@reset` |
| GET | `get-profile` | profile | `Api/ProfileController@getProfile` |
| GET | `get-profile-page` | profile | `ProfileController@getProfilePageData` |
| POST | `update-profile` | profile · EditProfileCubit | `ProfileController@updateProfile` |
| POST | `update-password` | profile · UpdatePasswordCubit | `ProfileController@updatePassword` |
| POST | `update-social` | profile | `ProfileController@updateSocial` |
| POST | `update-photo` | profile · EditProfileCubit | `ProfileController@updatePhoto` |
| POST | `update-location` | core/location | `ProfileController@updateLocation` (debounce سيرفري: يتجاهل لو <30م و<5د؛ يخزّن `user_locations` + cache user/IP) |
| POST | `update-fcm-token` | notifications | `ProfileController@updateFcmToken` |
| DELETE | `delete-profile` | profile · DeleteProfileCubit | `ProfileController@deleteProfile` |

## 02 — Home & Sliders ⬜
| GET | `home` | home · HomeCubit | `Api/HomeController` (بموقع: بلا cache؛ بلا موقع: 10د + version bump — e3cab23) |
| GET | `sliders` | home · SlidersCubit | `Api/HomeController`/SliderController |

## 03 — Trips ⬜
| GET | `prices` | trip | `Api/PriceController`/TripController |
| GET | `categories` | trip | `Api/CategoryController` |
| POST | `trips/prepare` · GET `trips/prepare/{token}` · GET `trips/prepare/{id}` | trip (v1) | `Api/TripController@prepare_trip` |
| POST | `trips/store` · GET `my-trips` · GET `view_trip/{id}` | trip (v1) | `TripController@save_trip/get_my_trips/view_trip` |
| POST | `trips/save-trip-to-email` | trip | `TripController` |
| POST | `v2/trips/prepare` · `v2/trips/reprepare` · GET `v2/trips/prepare/{token}` · GET `v2/trips/view/{token}` | trip (v2) | `Api/EnhancedTripV2Controller` |
| POST | `v2/trips/save` · GET `v2/trips/my-trips` · DELETE `v2/trips/{id}` | trip (v2) | `EnhancedTripV2Controller` |

## 04 — Places ⬜
| GET | `places` · `places/{slug}` | places | `Api/PlaceController@index/show` |
| GET | `places/places-data-for-map` | exploration | `PlaceController@placesDataForMap` |

## 05 — Stores ⬜
| GET | `stores` · `stores/{slug}` | stores | `Api/StoreController@index/show` |

## 06 — Restaurants (Zad) ⬜
| GET | `zads` · `zads/{slug}` | restaurants | `Api/ZadController@index/show` |
| GET | `zads/{id}/menus` | restaurants | `Api/MenuController` |
| GET | `zads/{id}/offers` | restaurants | `Api/OfferController` |

## 07 — Tasneef (unified listing) ⬜
| GET | `guides` · `places` · `zads` · `swalefs` · `stores` · `applications` | tasneef | نفس controllers الكيانات (index بفلاتر) |

## 08 — Exploration / Search ⬜
| GET | `global-search` | exploration | `Api/SearchController@globalSearch` |
| GET | `global-map-data` | exploration | `SearchController@global_map_data` |

## 09 — Events ⬜
| GET | `events/{slug}` | events | `Api/EventController` |

## 10 — Swalef & Stories ⬜
| GET | `swalefs/{slug}` | stories | `Api/SwalefController` |
| GET | `stories/myLastDayStories` | user_stories | `Api/StoryController@myLastDayStories` |

## 11 — Tour Guides ⬜
| GET | `get-guide` · `guides/{id}` · `guides` (filters) | tour_guide | `Api/GuideController@index/show` |
| POST | `store-guide` · `update-guide-photo` | tour_guide | `GuideController@storeGuide/updatePhoto` |
| GET | `languages` · `regions` | tour_guide/taxonomy | `Api/LanguageController`/`RegionController` |

## 12 — Landmarks ⬜
| GET | `landmark/list` · `landmark/show/{id}` · `landmark/myLandMarks` | landmarks | `Api/LandMarksController` |
| POST | `landmark/store` | landmarks | `LandMarksController@store` |

## 13 — Favorites & Saved ⬜
| GET | `get-my-favorites` · POST `favorites` | favorites | `Api/FavoriteController` |
| GET | `get-my-saved` | saved | `Api/SavedController` |

## 14 — Rates ⬜
| POST | `rates` | rates | `Api/RateController@store` |

## 15 — My Properties ⬜
| GET | `my-properties` · POST `properties` | my_properties | `Api/PropertiesController` |

## 16 — User Stories ⬜
| POST | `stories/store` · DELETE `stories/{id}/delete` | user_stories | `Api/StoryController@store/delete` |

## 18 — Taxonomy ⬜
| GET | `regions` · `cities` · `languages` · `prices` · `categories` | shared | `RegionController`/`CityController`/`LanguageController`/`PriceController`/`CategoryController` |

## Misc / Supporting ⬜
| GET | `sliders` · `contact-us` · `subscribe` · `settings` · `gallery` | various | `SettingController`/`ContactusController`/`SubscribeController`/`GalleryController` |

---
## Legend & Notes
- **v1/v2 Trips** يتعايشان؛ v2 = Enhanced (فترات صباح/مساء). v1 legacy.
- كل الردود بغلاف `{code/status, message, data}` من `ApiModalController`.
- Dashboard (web/session) غير مدرج هنا — راجع Feature 26 لاحقًا.
- أسماء بعض الـ controllers مُستنتجة من التحليل؛ تُثبَّت 100% عند تحليل كل ميزة.

_آخر تحديث: 2026-07-05._
