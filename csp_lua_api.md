# CSP Lua API — Assetto Corsa'ya ne kadar müdahale edebiliriz?

Kaynak: kurulu CSP **0.2.11 (build 3465)**, `extension/internal/lua-sdk/ac_apps/lib.lua` (17.475 satır). Bu dosya SDK'dan otomatik çıkarıldı; açıklamalar SDK'daki İngilizce metinlerdir. Hiçbir çağrı oyunda tek tek denenmedi.

## 1. Özet: müdahale seviyeleri

| Alan | Seviye | Not |
|---|---|---|
| Aracı konumlandırma / hız | **Tam** | `setCarPosition`, `setCarVelocity`, `teleportCarTo`, `addForce`, `setWheelAngularVelocity` |
| Girdiler (gaz/fren/debriyaj/el freni/direksiyon/vites) | **Tam** | `forceUser*For`, `overrideSteering`, `lockUserControlsFor`, `engageGear`, `setCarAutopilot` |
| Araç durumu (yakıt, hasar, lastik, sıcaklık, motor) | **Tam** | `setCarFuel`, `setCarBodyDamage`, `setTyresTemperature`, `blowTyres`, `setEngineRPM`... |
| Lastik grip, hava yoğunluğu, ağırlık, restriktör | **Geniş** | `setGripDecrease`, `setAirDensityMultiplier`, `setCarBallast`, `setCarRestrictor` |
| AI araçlar | **Çok geniş** | ~30 `setAI*` fonksiyonu: seviye, agresiflik, hız sınırı, spline kayması, konum, lastik, hata yapma |
| Yeni fizik nesneleri | **Var** | `physics.RigidBody`, `Collider.*`, `addTrackSurface`, `raycastTrack` |
| Çarpışma | **Var** | `setCarCollisions`, `disableCarCollisions` |
| Yarış kuralları (bayrak, ceza, DRS, pit) | **Var** | `overrideRacingFlag`, `setCarPenalty`, `setCarDRS`, `blockTeleportingToPits` |
| Kamera | **Tam** | `ac.grabCamera` ile kendi kameran (FastTravel haritası bunu kullanıyor) |
| Hava / ışık / saat | **Sınırlı** | Hava zamanı (`setWeatherExactUTC`, `setWeatherTimeMultiplier`, `setWeatherTimeOffset`), kar (`forceSnow`), farlar (`setHeadlights`, `setDaytimeLights`). Yağmur/bulut gibi hava değerlerini ayarlayan ayrı bir `ac.set*` yok; çoğu `ac.get*` ile okunur — bölüm 5 |
| Çizim / HUD / arayüz | **Tam** | `ui.*` (389 fonksiyon), `render.*` (38), kendi pencerelerin |
| Ses | **Var** | `ac.AudioEvent`, ses seviyesi/olaylar |
| Dosya / ağ | **Var** | `io.*`, `web.*` (HTTP istekleri), `os.*` |
| Pist (mesh/sahne düğümleri) | **Okuma + sınırlı değişiklik** | `ac.findNodes` ile sahne düğümleri, malzeme, görünürlük |

**Sınırlar** (SDK'dan çıkarılabilenler, deneyle doğrulanmadı):
- Çoğu `physics.*` çağrısı **tek kişilik / izinli oturumlarda** çalışır. Önce `physics.allowed()` ve gerekirse `ac.isCarResetAllowed()` kontrol edilmeli. Online sunucuda genelde kapalıdır.
- Fizik durumu (`ac.getCarPhysics`) **sadece oyuncu aracı için**; replay ve uzak araçlarda fizik alanları yoktur.
- `physics.*` değişikliklerinin çoğu **turu geçersiz kılar** (`invalidates current lap`).
- `engageGear` H-shifter'lı oyuncu aracında iyi çalışmayabilir.
- `ac.loadCarState` sadece offline ve basit oturum modlarında kullanılabilir.
- `overrideSteering`'in değer birimi dokümanda belirtilmemiş. [Unknown]
- Lua uygulamaları **Python (`apps/python`) tarafından ayrı** çalışır; ikisi arasında doğrudan çağrı yok (dosya/ağ ile köprü gerekir).

## Pratik karar rehberi: hedef → veri → API

Bu dokümanı bir fonksiyon kataloğu gibi değil, üç soruyla kullan: **ne görmek istiyorum, neyi değiştirmek istiyorum ve bunun oturum kuralı nedir?** Önce durumu oku, sonra gerekiyorsa ekrana çiz veya değişiklik yap. Araç fiziğine dokunan çağrıları en son aşama olarak düşün.

### Temel veri yapıları — nereden başlanır?

| Veri / yapı | Ne anlatır? | Ne zaman kullanılır? |
|---|---|---|
| `ac.getCar(i)` → `ac.CarState` | Bir aracın konumu, hızı, girdileri, turu, hasarı, sistemleri ve görsel durumu. `i = 0` normalde ilk/oyuncu aracıdır. | Telemetri, HUD, araç karşılaştırma, ghost kaydı ve araç durumunu gösterme. |
| `ac.getCarPhysics(i)` | Oyuncu aracının daha ayrıntılı fizik anlık görüntüsü. Uzak araç/replay için alanları olmayabilir. | Süspansiyon, lastik, kuvvet veya ayrıntılı sürüş analizi gerektiğinde. |
| `ac.getSim()` → `ac.SimState` | Oturum, pist, hava, kamera, zaman ve genel simülasyon durumu. | Yağmurda farklı HUD, yarış durumuna göre davranış veya oturum sayacı. |
| `car.wheels[i]` | `ac.getCar()` içindeki dört tekerleğin görsel/telemetri durumu. | Lastik sıcaklığı, aşınma, basınç ya da tekerlek bazlı gösterge. |
| `getCarPhysics(i).wheels[i]` | Tekerleğin fizik tarafındaki ek verileri. | Tutunma ve süspansiyon davranışını daha ayrıntılı analiz etmek. |
| `vec3` | Üç bileşenli uzaysal değer: ör. `position`, `velocity`, `look`, `up`. | Dünya koordinatında işaret, çizim, kamera veya hız yönü işlemleri. |
| `mat4x4` | Konum ve yönelimi birlikte taşıyan dönüşüm matrisi: ör. `transform`, `bodyTransform`. | Bir 3D nesneyi araca/piste hizalama; yalnızca konum gerekiyorsa önce `position` ile başla. |

Örnek okuma akışı: `local car = ac.getCar(0)` ile anlık araç kaydını al; ardından `car.speedKmh`, `car.position`, `car.gas`, `car.brake`, `car.lapTimeMs` veya `car.wheels[0]` gibi ihtiyaç duyduğun alanları oku. Genel koşullar için ayrı olarak `local sim = ac.getSim()` al; örneğin `sim.rainIntensity`, `sim.isOnlineRace`, `sim.sessionTimeLeft`.

> **Kural:** `ac.getCar()` ve `ac.getSim()` gözlem için en güvenli başlangıçtır. `physics.*` ile araç durumunu değiştirmek ise ayrı izin, offline/izinli oturum ve tur geçersizliği riski taşır.

### Hangi amaç için hangi API ailesi?

| Hedef | Önce oku | Sonra kullan | Tipik sonuç |
|---|---|---|---|
| Sürüş telemetrisi / dashboard | `ac.getCar(0)`, `ac.getSim()` | `ui.*`, gerekirse `render.*` | Hız, vites, delta, lastik ve hava bilgisi gösteren uygulama. |
| Tur analizi / veri kaydı | `ac.getCar(0)`, `ac.getCarPhysics(0)`, tekerlekler | `io.*` ile kayıt, `web.*` ile gönderim | Her karede örnek alıp CSV/JSON veya yerel API'ye telemetri göndermek. |
| Ghost / geri sarma prototipi | `position`, `velocity`, `gear`, `rpm`, direksiyon ve tekerlek durumu | `physics.setCarPosition`, `setCarVelocity`, `engageGear`, `setEngineRPM` | Kaydedilmiş bir anı yeniden oynatmak; yalnızca izinli fizik bağlamında. |
| Antrenman senaryosu | `ac.getCar(0)` ile başlangıcı doğrula | Konum/hız/vites/yakıt/lastik için ilgili `physics.set*` çağrıları | Aracı belirli viraj çıkışına veya egzersize hazırlamak. |
| Kendi kamera / sinematik | `ac.getSim()` ve kamera vektörleri | `ac.grabCamera`, kamera örneğinin konum/yön/FOV metotları | Harita, chase-cam veya özgür kamera deneyimi. |
| Pist üstü 3D işaret / HUD | Araç konumu, kamera ve pist bilgisi | `render.*`, `ui.*`, gerekirse `ac.findNodes` | Fren noktası göstergesi, rota oku, performans paneli. |
| Yarış kuralı veya etkinlik | `ac.getSim()`, araç tur/ceza durumu | `overrideRacingFlag`, `setCarPenalty`, `setCarDRS`, pit çağrıları | Senaryo bazlı bayrak, ceza, DRS veya pit akışı. |
| AI davranışı | AI aracın `ac.getCar(i)` durumu | `physics.setAI*` fonksiyonları | Hız, agresiflik, hata ve spline davranışı ayarlanan AI. |
| Dış uygulamayla haberleşme | Okunacak telemetri nesneleri | `web.*`, `io.*`, `os.*` | Racegrid gibi yerel servise veri gönderme veya komut alma. |

### Güvenli çalışma sırası

1. **Oku:** İhtiyacın olan en küçük veri kümesini `ac.getCar(0)` ve/veya `ac.getSim()` ile al.
2. **Karar ver:** Koşulları yalnızca bu anlık görüntüden üret; örneğin hız 80 km/h üstündeyse uyarı ver.
3. **Göster veya kaydet:** Değişiklik gerekmiyorsa `ui.*`, `render.*`, `io.*` veya `web.*` ile burada bitir.
4. **Değiştirmen gerekiyorsa kontrol et:** `physics.allowed()` ve gerekli durum kontrollerini yap; online oturum ve tur geçersizliği etkisini hesaba kat.
5. **Tek amaca yönelik çağrı seç:** Örneğin hız vermek için yalnızca `setCarVelocity`; konum + yön için `setCarPosition`; tam kamera kontrolü için `grabCamera` kullan. Birden çok çağrıyı ancak aynı fizik anını tutarlı kurmak gerektiğinde birleştir.

### Başlangıç için üç küçük proje

- **Lastik ve fren HUD'ı:** `ac.getCar(0)` + `car.wheels[i]` oku, `ui.*` ile dört lastiğin sıcaklığını ve `car.brake` değerini göster. Fizik yazma yoktur; en güvenli ilk proje budur.
- **Viraj giriş telemetrisi:** Her güncellemede `speedKmh`, `brake`, `steer`, `position` ve `lapTimeMs` kaydet; `io.*` ile dosyaya veya `web.*` ile yerel Racegrid servisine gönder. Sonradan viraj başına frenleme noktalarını çıkarabilirsin.
- **Başlangıç noktası egzersizi:** Önce offline/izinli fizik erişimini doğrula. Sonra `setCarPosition`, `setCarVelocity` ve `engageGear` ile belirli bir senaryoyu başlat. Her fizik yazımını açık bir kullanıcı eylemine bağla.

## 2. Mevcut uygulamalar için hızlı reçeteler

- **Işınlanma + hız/vites/direksiyon koruma:** `setCarPosition` + `setCarVelocity` + `engageGear` + `setEngineRPM` + `setWheelAngularVelocity` + `overrideSteering` (bu repodaki `fasttravel_app` bunu yapıyor).
- **Gerçek durum kaydı:** `ac.saveCarStateAsync` / `ac.loadCarState` (tam fizik anlık görüntüsü, offline).
- **Hayalet/geri sarma:** her karede `getCar(0)` durumunu kaydet, sonra `setCarPosition/Velocity` ile oynat.
- **Antrenman senaryoları:** başlangıç hızı + vites + konumla "viraj çıkışı" tekrarları, `setCarFuel`/lastik ısısı sıfırlama.
- **Dış kontrol:** `web.*` ile yerel bir sunucuya (ör. racegrid API) telemetri gönder veya komut çek.
- **Telemetri:** 266 alanlı `ac.getCar` + 16 alanlı `getCarPhysics` + tekerlek alanları — sürüş verisinin tamamı okunabilir.

## 3. `physics.*` — fizik müdahalesi (tam liste, 94 fonksiyon)

- **`physics.getPointVelocity(carIndex, position, posLocal)`** — Returns velocity (in world coordinates) of a car point. If there is no such car or physics currently available, returns vector with zeroes.
- **`physics.getExtendedSuspensionTravel(carIndex, corner)`** — 
- **`physics.getExtendedDamperTravel(carIndex, corner)`** — 
- **`physics.Collider.Box(size, offset, look, up, debug)`** — Box collider.
- **`physics.Collider.Sphere(radius, offset, debug)`** — Sphere collider.
- **`physics.Collider.Capsule(length, radius, offset, look, debug)`** — Capsule collider (like cylinder, but instead of flat caps it has hemispheres and works a bit faster).
- **`physics.Collider.Cylinder(length, radius, offset, look, debug)`** — Cylinder collider (slower than capsule, consider using capsule where appropriate).
- **`physics.RigidBody(collider, mass, cog, semiDynamic, startsInWorld)`** — Create a new physics rigidbody with a collider (could be a mesh from KN5 or one or several geometric colliders). It’s up to your code to keep track of rigidbody transform and assign it to something visual. Example (link to “cone.kn5”: https://files.acstuff.club/VmRC/cone.kn5; note how single KN5 stores both visuals and a collider mesh): local targetRoot = ac.findNodes('dynamicRoot:yes') local loadedModel = targetRoot:loadKN5({filename = 'cone.kn5', filter = 'mesh'}) local physics = physics.RigidBody({   filename = 'cone.kn5',   filter = 'collider',   debug = true,   transform = true }, 10) physics:setSemiDynamic(true, true) physics:setTransformation(mat4x4.translation(vec3(-2679.85, -41.06, -202.68))) setInterval(function ()   loadedModel:setTransformationFrom(physics) end) By default spawened bodies have a bit of a damping. Use `:setDamping()` to remove it.
- **`physics.addTrackSurface(vertices, indices, surfaceName, transform, debugColor)`** — Create a new track physics mesh (could be a wall or a road). Requires AC to use Embree for raycasting (default settings). Currently, causes a huge lag, working on solving the problem. Not available to scripts without Physics API access.
- **`physics.teleportCarTo(carIndex, spawnSet)`** — Teleports car to certain spawn point, invalidates current lap.
- **`physics.setCarPosition(carIndex, pos, dir)`** — Sets car position and orientation and invalidates current lap time. Car will be aligned on track surface.
- **`physics.setCarVelocity(carIndex, velocity)`** — Sets car velocity and invalidates current lap time.
- **`physics.engageGear(carIndex, gearToEngage)`** — Forces certain gear to engage. Might not work well for user car with H-shifter. Also, invalidates current lap time.
- **`physics.setMGUKDelivery(carIndex, level)`** — Sets MGUK delivery mode. Note: for controling user car consider using `ac.setMGUKDelivery()` instead, it’ll work better with HUD and everything.
- **`physics.setMGUKRecovery(carIndex, level)`** — Sets MGUK recovery mode. Note: for controling user car consider using `ac.setMGUKDelivery()` instead, it’ll work better with HUD and everything.
- **`physics.setGentleStop(carIndex, stop)`** — Activates or deactivates gentle stopping.
- **`physics.lockUserControlsFor(time)`** — Locks user controls for specified time. Use this one instead of `physics.setCarNoInput()` if you only need to stop car temporarily and then let user control it back.
- **`physics.lockUserGearboxFor(time, resetGear)`** — Locks user car gearbox for specified time. Uses the same logic AC uses for starting line.
- **`physics.forceUserBrakesFor(time, brakingIntensity)`** — Forces fully engaged brakes for user car for given time. Invalidates lap. Subsequent calls override value set earlier.
- **`physics.forceUserThrottleFor(time, throttleIntensity)`** — Forces fully engaged throttle pedal for user car for given time. Invalidates lap. Subsequent calls override value set earlier.
- **`physics.forceUserClutchFor(time, clutchValue)`** — Forces fully engaged clutch pedal for user car for given time. Invalidates lap. Subsequent calls override value set earlier.
- **`physics.forceUserHandbrakeFor(time, handbrakeValue)`** — Forces fully engaged handbrake for user car for given time. Invalidates lap. Subsequent calls override value set earlier.
- **`physics.offsetUserSteering(offset)`** — Offsets user steering angle.
- **`physics.overrideUserSteering(value, share)`** — Overrides user steering angle.
- **`physics.overrideSteering(carIndex, value)`** — Overrides steering completely for a certain car.
- **`physics.resetCarState(carIndex, fuelMult)`** — Resets car state (repairs damage, resets fuel) and invalidates current lap.
- **`physics.setCarFuel(carIndex, fuelLiters)`** — Sets car fuel and invalidates current lap.
- **`physics.setCarNoInput(active)`** — Disables input for user car, switches car to “parked” state. If it’s meant for game rules and such, try `physics.lockUserControlsFor()` instead.
- **`physics.setCarAutopilot(active, chooseTyres)`** — Activates or deactivates autopilot for user car. Invalidates lap.
- **`physics.getCarInputControls()`** — Returns state of car controls user input. Works even if user car controls were deactivated or AI is controlling user car.
- **`physics.setGripDecrease(carIndex, wheels, decrease)`** — Reduces grip for some tyres of a certain car. Invalidates current lap. Could be used as a cheap way to simulate some things like ice.
- **`physics.setTyresAIMultiplier(carIndex, wheels, aiMultiplier)`** — Not recommended to use. Invalidates lap.
- **`physics.setAirDensityMultiplier(carIndex, value)`** — Changes air density for a certain car. Invalidates lap.
- **`physics.resetWearPlanks(carIndex)`** — Resets wear planks. Invalidates lap.
- **`physics.resetTrackPhysicsObject(filter)`** — Resets a position of track physics objects.
- **`physics.blowTyres(carIndex, wheels)`** — Blows up some tyres of a certain car. Does not invalidate current lap.
- **`physics.setTyresBlankets(carIndex, wheels, blanketActive)`** — Sets tyre blankets and invalidates current lap. If set, they would keep tyres at optimal temperature until car would reach 10 km/h, and then they would be removed. If you need to set tyres temperature for a moving car, use `physics.setTyresTemperature()` (to get optimal tyre temperature, use `ac.getCar(N).wheels[0].tyreOptimumTemperature`.
- **`physics.setTyresCompound(carIndex, wheels, compoundIndex)`** — Sets tyre compounds, circumventing the current setup. Use carefully in very special cases, as a last resort. Might cause physics to explode.
- **`physics.setWaterTemperature(carIndex, temperature)`** — Sets water temperature. Does not invalidate lap.
- **`physics.setTyresTemperature(carIndex, wheels, temperature, destinationFlags)`** — Sets tyres temperature and invalidates current lap. To get optimal tyre temperature, use `ac.getCar(N).wheels[0].tyreOptimumTemperature`.
- **`physics.setTyresVirtualKM(carIndex, wheels, virtualKm)`** — Sets driven distance used for wear computation.
- **`physics.setCarBodyDamage(carIndex, bodyDamage)`** — Sets damage levels of a given car, invalidates lap if car body state improves.
- **`physics.setCarEngineLife(carIndex, engineLife)`** — Sets damage level of the engine of a given car, invalidates lap if engine state improves.
- **`physics.setCarBallast(carIndex, ballast)`** — Sets car ballast, invalidates current lap if ballast is negative.
- **`physics.setCarRestrictor(carIndex, restrictor)`** — Sets car restrictor. Does not invalidate lap.
- **`physics.setCarAutoShifter(carIndex, active)`** — Activates or deactivates automatic shifter for a car. Does not invalidate lap.
- **`physics.setCarCollisions(carIndex, disable)`** — Enables and disables car collisions (wouldn’t affect disabled collisions in pits, more like an extra option to  further disable collisions). Invalidates lap.  Use `physics.disableCarCollisions()` instead.
- **`physics.disableCarCollisions(carIndex, disable, applyToRemote)`** — Enables and disables car collisions (wouldn’t affect disabled collisions in pits, more like an extra option to  further disable collisions). Invalidates lap of the car it’s applied to.  By default won't affect remote cars (a bug, kept for backwards compatibility). Set third argument (added in 0.2.8) to `true`  to change that behavior.
- **`physics.resetColliderWears(carIndex)`** — Resets collider wears.
- **`physics.allowCarDRS(carIndex, active)`** — Allows and disallows car DRS. If DRS is already disallowed by track zones, wouldn’t override it, but allows to add more  conditions disallowing DRS further (FIA rules and all that). If DRS is already engaged, it’ll be disengaged. Does not  invalidate lap.
- **`physics.setCarDRS(carIndex, active)`** — If DRS is present and available, activates or deactivates it. Does not invalidate lap.
- **`physics.setCarPenalty(penaltyType, penaltyParam)`** — Sets penalty for user car, invalidates current lap.
- **`physics.setAllowedTyresOut(allowedTyresOut)`** — New Mode scripts can use this function even if the rest of Physics API is not available.
- **`physics.awakeCar(carIndex)`** — Stops car from sleeping, invalidates current lap.
- **`physics.addForce(carIndex, position, posLocal, force, forceLocal, wheelIndex)`** — Adds force to a car, invalidates current lap. Note: you have to scale your force by `dt` accordingly.
- **`physics.addWheelTorque(carIndex, wheels, torque)`** — Alters angular speed of a wheel or several. Invalidates current lap.
- **`physics.setWheelAngularVelocity(carIndex, wheels, angularVelocity)`** — Alters angular speed of a wheel or several. Invalidates current lap.
- **`physics.markLapAsSpoiled(carIndex)`** — Marks current lap as spoiled. Works only during the race
- **`physics.setEngineStallEnabled(carIndex, value)`** — Enables engine stalling (if RPM is below idling, torque would be added only if backspace is pressed). Use it together with `physics.setEngineRPM()` to shut off the engine.
- **`physics.setEngineRPM(carIndex, rpm)`** — Sets engine RPM.
- **`physics.setDriverName(name, nationCode)`** — Sets name of a user, announces new name online to other clients. Could be used for some hot driver swapping.
- **`physics.setDriverTeam(team)`** — Sets name of a user’s team, announces new name online to other clients. Could be used for some hot driver swapping.
- **`physics.overrideRacingFlag(flag)`** — Overrides racing flag. Don’t forget to call `physics.overrideRacingFlag(ac.FlagType.None)` later to revert to original AC flag handling.
- **`physics.blockTeleportingToPits()`** — Prevents car from teleporting to pits no matter what, apart from starting new session, by EMERGENCY_RESET option or with physics API. Make sure to release disposable when you no longer need the block.
- **`physics.startPhysicsWorker(script, data, callback)`** — Load a separate Lua script and run it in physics thread. Works similar to `ac.startBackgroundWorker()`, but these workers run at 333 frames a second and have access to some very limited physics API. Use carefully, make sure to keep scripts as lightweight as possible. Good if you want to simulate some continuous force. Make sure to use custom `physics.` methods to access latest physics state as well. Important: use this script to access some limited physics API such as `physics.addForce()`. Sim state (`ac.getSim()`, `ac.getCar()`, etc.) still updates at render rate. To access current car positions at physics rate, use `physics.getBodyTransform()`. Be extremely careful with these types of scripts: slow code will mess up physics performance! This type of worker still relies on `script.update()`. Call `worker.terminate()` to stop it. No other `worker` methods are available. Use shared structures to exchange data between your main script and a worker.
- **`physics.allowed()`** — Returns `true` if script can affect physics engine. If not, only `physics.raycastTrack()` is available.
- **`physics.raycastTrack(pos, dir, length, point, normal)`** — Casts a ray from given position in a certain direction to check if it hits any physics surfaces. Returns distance to a hit, or -1 if there wasn’t any. Optionally can return contact point and normal too, if vectors in their place are passed. Note: smaller rays are faster to cast. Also, rays going straight down are slightly faster as well.
- **`physics.setAILevel(carIndex, level)`** — Changes AI level. Use `ac.getCar(carIndex).aiLevel` to get current value.
- **`physics.setAINoInput(carIndex, active, stall)`** — Disables input for an AI car, switches car to “parked” state.
- **`physics.setAIDriverName(carIndex, name, nationCode)`** — Sets name of an AI, announces new name online to other clients. Could be used for some hot driver swapping.
- **`physics.setAIDriverTeam(carIndex, team)`** — Sets name of an AI’s team, announces new name online to other clients. Could be used for some hot driver swapping.
- **`physics.setAICarPosition(carIndex, pos, dir)`** — Changes position of an AI car.
- **`physics.setAIAggression(carIndex, aggression)`** — Changes AI aggression. Use `ac.getCar(carIndex).aiAggression` to get current value.
- **`physics.setAIAeroHint(carIndex, value)`** — Changes AI `[PHYSICS_HINTS] AERO_HINT` value.
- **`physics.setAITyresHint(carIndex, value)`** — Changes AI `[TYRES] HINT` value.
- **`physics.preventAIFromRetiring(carIndex)`** — Prevent an AI from retiring for some time. If not moving, they will still retire some time after this function is called, unless you’ll keep calling it.
- **`physics.setAIShiftingThresholds(carIndex, upshift, downshift)`** — Changes AI shifting thresholds. Once called, stops CSP logic altering thresholds dynamically.
- **`physics.setAICaution(carIndex, caution)`** — Changes AI caution, altering the distance it keeps from the car in front of it. Default value: `1`. Experimental.
- **`physics.setAILookaheadBase(carIndex, value)`** — Changes AI base lookahead value. Use `ac.INIConfig.carData(carIndex, 'ai.ini'):get('LOOKAHEAD', 'BASE', 12)` to load original value first and go from there (don’t re-read it each frame though, it’ll get expensive).
- **`physics.setAILookaheadGasBrake(carIndex, value)`** — Changes AI gas/brake lookahead value. Use `ac.INIConfig.carData(carIndex, 'ai.ini'):get('LOOKAHEAD', 'GAS_BRAKE_LOOKAHEAD', 10)` to load original value first and go from there (don’t re-read it each frame though, it’ll get expensive).
- **`physics.setAIBrakeHint(carIndex, value)`** — Changes AI brake hint value. Use `ac.INIConfig.carData(carIndex, 'ai.ini'):get('PEDALS', 'BRAKE_HINT', 1)` to load original value first and go from there (don’t re-read it each frame though, it’ll get expensive).
- **`physics.setAIStopCounter(carIndex, time)`** — Forces AI to brake for a specified amount of time. Originally, this mechanism is used to get AIs to brake after an incident. Subsequent calls overwrite current waiting, pass 0 to get cars to move.
- **`physics.setAIThrottleLimit(carIndex, limit)`** — Limits AI throttle pedal.
- **`physics.setAITopSpeed(carIndex, limit)`** — Limits AI top speed. Use `math.huge` (or just 1e9) to remove limitation.
- **`physics.setAISteerMultiplier(carIndex, mult)`** — Sets steering multiplier allowing to get cars to drive more straight in some cases.
- **`physics.setAISplineOffset(carIndex, offset, overrideAiAwareness)`** — Sets lateral offset along spline, can be used to get cars to drive further to the left or to the right. Actual offset will be limited by track width as well.
- **`physics.setAISplineAbsoluteOffset(carIndex, offset, overrideAiAwareness)`** — Sets lateral offset along spline, can be used to get cars to drive further to the left or to the right. Actual offset will be limited by track width as well.
- **`physics.setAISpline(carIndex, filename)`** — If not loaded yet, loads new AI spline in background and uses it instead of main API spline. Experimental.
- **`physics.setExtraAIGrip(carIndex, value)`** — Changes extra AI grip (120% by default).
- **`physics.setAIPitStopRequest(carIndex, value)`** — Controls if AI goes to pits or not.
- **`physics.setAITyres(carIndex, index)`** — Sets currently selected tyres, resetting damage as well.
- **`physics.setAIBrakeMistake(carIndex, time)`** — Triggers brake mistake causing wheels to lock on braking.
- **`physics.setAITractionMistake(carIndex, time, severity)`** — Triggers traction mistake.
- **`physics.setAIRainTyres(carIndex, useRainTyres)`** — Hint AIs to put rain tyres on or off, overrides default behaviour.

## 4. Araç durumu okuma

### `ac.getCar(i)` — görsel/grafik durumu (266 alan)

`mass`, `graphicsOffset`, `graphicsPitchRotation`, `graphicsToPhysicsTransform`, `physicsToGraphicsTransform`, `steerLock`, `maxFuel`, `exposureOutside`, `exposureInside`, `shakeMultiplier`, `headlightsColor`, `customCarColor`, `aabbCenter`, `aabbSize`, `index`, `gearCount`, `turboCount`, `tractionType`, `enginePosition`, `brakesBiasLimitDown`, `brakesBiasLimitUp`, `brakesBiasStep`, `maxRelativePlankWear`, `year`, `hShifter`, `adjustableTurbo`, `brakesCockpitBias`, `extendedPhysics`, `isRacingCar`, `isRallyCar`, `isDriftCar`, `isOpenWheeler`, `isEngineDiesel`, `isKunosCar`, `prefersImperialUnits`, `headlightsAreHeadlights`, `hasLowBeams`, `isLeftHandDrive`, `sirenHorn`, `hasFlashingLights`, `hasCockpitSwitchForUserSpeedLimiter`, `hasHornAudioEvent`, `hasAnalogTelltale`, `tractionControlModes`, `absModes`, `fuelMaps`, `tractionControl2Modes`, `hasEngineBrakeSettings`, `pitTransform`, `timestamp`, `transform`, `bodyTransform`, `worldToLocal`, `position`, `velocity`, `acceleration`, `angularVelocity`, `localAngularVelocity`, `localVelocity`, `up`, `look`, `side`, `driverEyesPosition`, `groundNormal`, `groundDistance`, `gas`, `brake`, `clutch`, `steer`, `handbrake`, `gear`, `engagedGear`, `fuel`, `rpm`, `rpmLimiter`, `rpmMinimum`, `speedKmh`, `speedMs`, `turboBoost`, `drivetrainSpeed`, `waterTemperature`, `minHeight`, `restrictor`, `ballast`, `ping`, `cgHeight`, `wheelsOutside`, `engineLifeLeft`, `dirt`, `damage`, `gearboxDamage`, `nodeIndex`, `visibleIndex`, `activeLOD`, `distanceToCamera`, `splinePosition`, `driftPoints`, `driftInstantPoints`, `driftComboCounter`, `collisionDepth`, `collisionPosition`, `collidedWith`, `lapCutsCount`, `lastLapCutsCount`, `aiLevel`, `aiAggression`, `aiRainTyres`, `currentSplits`, `lastSplits`, `bestSplits`, `bestLapSplits`, `wheels`, `extrapolatedMovement`, `isActive`, `isConnected`, `isRemote`, `isAIControlled`, `isUserControlled`, `isLapValid`, `isLastLapValid`, `isCameraOnBoard`, `isInPitlane`, `isInPit`, `isRetired`, `isEngineLimiterOn`, `isGearGrinding`, `hasTurningLights`, `headlightsActive`, `brakeLightsActive`, `flashingLightsActive`, `hornActive`, `focused`, `focusedOnInterior`, `useInternalAudio`, `isHidingLabels`, `isDriftBonusOn`, `isDriftValid`, `isRaceFinished`, `isDriverVisible`, `isDriverDoorOpen`, `isDriverSeatbeltOn`, `hazardLights`, `turningLeftLights`, `turningRightLights`, `turningLeftOnly`, `turningRightOnly`, `lowBeams`, `highBeams`, `extraA`, `extraB`, `extraC`, `extraD`, `extraE`, `extraF`, `extraG`, `extraH`, `extraI`, `extraJ`, `extraK`, `extraL`, `extraM`, `extraN`, `extraO`, `extraP`, `extraQ`, `extraR`, `extraS`, `extraT`, `kersCharging`, `turningLightsActivePhase`, `anyFlamesActive`, `justJumped`, `resetCounter`, `kersCharge`, `kersInput`, `kersCurrentKJ`, `kersMaxKJ`, `kersLoad`, `distanceDrivenTotalKm`, `distanceDrivenSessionKm`, `poweredWheelsSpeed`, `batteryVoltage`, `oilPressure`, `oilTemperature`, `exhaustTemperature`, `wiperModes`, `wiperMode`, `wiperSelectedMode`, `wiperSpeed`, `wiperProgress`, `bodyWetness`, `compass`, `lapTimeMs`, `bestLapTimeMs`, `previousLapTimeMs`, `lapCount`, `currentSector`, `previousSectorTime`, `racePosition`, `drivenInRace`, `estimatedLapTimeMs`, `performanceMeter`, `performanceMeterSpeedDifferenceMs`, `sessionLapCount`, `compoundIndex`, `sessionID`, `physicsAvailable`, `speedLimiterInAction`, `absInAction`, `tractionControlInAction`, `hasUserBrakeBias`, `hasCockpitMGUHMode`, `hasCockpitERSDelivery`, `hasCockpitERSRecovery`, `drsPresent`, `drsAvailable`, `drsActive`, `kersPresent`, `kersHasButtonOverride`, `kersButtonPressed`, `mguhChargingBatteries`, `manualPitsSpeedLimiterEnabled`, `userSpeedLimiterEnabled`, `autoShift`, `autoClutch`, `mgukDelivery`, `mgukDeliveryCount`, `mgukRecovery`, `tractionControlMode`, `absMode`, `engineBrakeSettingsCount`, `currentEngineBrakeSetting`, `fuelPerLap`, `differentialPreload`, `differentialCoast`, `differentialPower`, `awdFrontShare`, `awdCenterLock`, `drivetrainTorque`, `drivetrainPower`, `brakeBias`, `speedLimiter`, `turboBoosts`, `turboWastegates`, `tractionControl2`, `fuelMap`, `steerTorque`, `ffbSteerAssist`, `ffbBase`, `ffbFinal`, `ffbPure`, `ffbMultiplier`, `aeroLiftFront`, `aeroLiftRear`, `aeroDrag`, `caster`, `rideHeight`, `p2pStatus`, `p2pActivations`, `altitude`, `ambientOcclusion`, `rawAmbientOcclusion`, `carCamerasCount`, `currentPenaltyType`, `currentPenaltyParameter`, `extraIconsCount`

<details><summary>Açıklamalar</summary>

- `mass` (number) Car mass in kg.
- `graphicsOffset` (vec3) Graphics offset from car.ini
- `graphicsPitchRotation` (number) Graphics pitch rotation in radians from car.ini
- `graphicsToPhysicsTransform` (mat4x4) Graphics-to-physics transform based on graphics offset and pitch rotation from car.ini
- `physicsToGraphicsTransform` (mat4x4) Physics-to-graphics transform based on graphics offset and pitch rotation from car.ini
- `steerLock` (number) Maximum steering wheel angle in degrees.
- `maxFuel` (number) Maximum amount of fuel in liters.
- `exposureOutside` (number) Outboard exposure from car.ini
- `exposureInside` (number) Onboard exposure from car.ini
- `shakeMultiplier` (number) SHAKE_MUL value from car.ini
- `headlightsColor` (rgb) 
- `customCarColor` (rgb) Custom car color selected online, or `math.nan` if color is the default one
- `aabbCenter` (vec3) Center of AABB (calculated from LOD D or collider mesh).
- `aabbSize` (vec3) Size of AABB in meters (calculated from LOD D or collider mesh).
- `index` (integer) 0-based (0 for first car).
- `gearCount` (integer) Excluding neutral and reverse. Physics-only (see `ac.CarState.physicsAvailable`)
- `turboCount` (integer) Physics-only (see `ac.CarState.physicsAvailable`)
- `tractionType` (integer) 0 for rwd, 1 for fwd, 2 for awd, 3 for new awd, -1 for N/A. Physics-only (see `ac.CarState.physicsAvailable`)
- `enginePosition` (integer) 0 for unspecified, 1 for front, 2 for rear, 3 for mid. Physics-only (see `ac.CarState.physicsAvailable`)
- `brakesBiasLimitDown` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `brakesBiasLimitUp` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `brakesBiasStep` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `maxRelativePlankWear` (number) From 0 to 1. Physics-only (see `ac.CarState.physicsAvailable`), 0 if not available.
- `year` (integer) Manufactoring year.
- `hShifter` (boolean) True if car has H-shifter in its physics data.
- `adjustableTurbo` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `brakesCockpitBias` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `extendedPhysics` (boolean) True if extended car physics is active.
- `isRacingCar` (boolean) True for racing cars (cars with class different from “road” or “street”).
- `isRallyCar` (boolean) Car counts as a rally car if it has a corresponding tag or “rally” in its name.
- `isDriftCar` (boolean) Car counts as a drift car if it has a corresponding tag or “drift” in its name.
- `isOpenWheeler` (boolean) Check is based on car tags.
- `isEngineDiesel` (boolean) Check is based on car tags.
- `isKunosCar` (boolean) True if car is standard, from Kunos.
- `prefersImperialUnits` (boolean) True for cars from UK or USA.
- `headlightsAreHeadlights` (boolean) If set to `false`, headlights do something else (like change display mode).
- `hasLowBeams` (boolean) If set to `false`, low beams are not present (as in, headlights operate in a single mode).
- `isLeftHandDrive` (boolean) If set to `true`, steering wheel is on the left side.
- `sirenHorn` (boolean) If set to `true`, horn is working in siren mode (press to toggle).
- `hasFlashingLights` (boolean) 
- `hasCockpitSwitchForUserSpeedLimiter` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `hasHornAudioEvent` (boolean) 
- `hasAnalogTelltale` (boolean) Car has max RPM arrow.
- `tractionControlModes` (integer) 0 if TC is not present. Physics-only (see `ac.CarState.physicsAvailable`)
- `absModes` (integer) 0 if ABS is not present. Physics-only (see `ac.CarState.physicsAvailable`)
- `fuelMaps` (integer) 
- `tractionControl2Modes` (integer) 
- `hasEngineBrakeSettings` (boolean) 
- `pitTransform` (mat4x4) Transformation of pit position.
- `timestamp` (number) Time of last physics state record, in milliseconds (counting from the same point as ac.SimState.time)
- `transform` (mat4x4) Car physics transformation in world space (does not match body transformation! for it, use `bodyTransform`)
- `bodyTransform` (mat4x4) Car visual transformation (the one applied to 3D model).
- `worldToLocal` (mat4x4) Inverse of car visual transformation.
- `position` (vec3) Car position in the world (corresponds to 0 coordinate in its model space).
- `velocity` (vec3) Car velocity in m/s.
- `acceleration` (vec3) G-forces, X for sideways relative to car, Z for forwards/backwards.
- `angularVelocity` (vec3) 
- `localAngularVelocity` (vec3) 
- `localVelocity` (vec3) 
- `up` (vec3) Vector facing upwards (normalized).
- `look` (vec3) Vector facing forward (normalized).
- `side` (vec3) Vector facing sideways (normalized).
- `driverEyesPosition` (vec3) In-car coordinates for driver eyes position (can be changed by user).
- `groundNormal` (vec3) Normal of ground below car.
- `groundDistance` (number) Distance of ground below car (defines ground plane together with normal).
- `gas` (number) Throttle, from 0 to 1.
- `brake` (number) Brake, from 0 to 1.
- `clutch` (number) Clutch, from 0 to 1 (1 for pedal fully depressed).
- `steer` (number) Angle of steering wheel in degrees.
- `handbrake` (number) Handbrake, from 0 to 1.
- `gear` (integer) Current gear, 0 for neutral, -1 for reverse. Does not go through 0 on sequential shifts
- `engagedGear` (integer) Current gear, 0 for neutral, -1 for reverse. Goes through 0 on sequential shifts
- `fuel` (number) Remaining fuel in liters.
- `rpm` (number) Engine RPM.
- `rpmLimiter` (number) RPM limiter threshold, if exists.
- `rpmMinimum` (number) Minimum RPM (or -1 if physics not available).
- `speedKmh` (number) Current speed in km/h.
- `speedMs` (number) Current speed in m/s.
- `turboBoost` (number) Turboboost value, from 0 and upwards.
- `drivetrainSpeed` (number) Speed delivered to wheels.
- `waterTemperature` (number) Approximation of water temperature in °C done by original AC.
- `minHeight` (number) Minimum allowed ride height in meters.
- `restrictor` (number) Restrictor.
- `ballast` (number) Ballast in kg.
- `ping` (integer) Ping in ms (or -1 if not available).
- `cgHeight` (number) 
- `wheelsOutside` (integer) Number of wheels outside of allowed area.
- `engineLifeLeft` (number) Engine life left (1000 for new engine, breaks at 0).
- `dirt` (number) Body dirt level from 0 to 1.
- `damage` (number[]) Damage values from 0 to maximum collision speed in km/h for four different zones (fifth one is not really used). 5 items, starts with 0.
- `gearboxDamage` (number) Gearbox damage (0 for new gearbox, 1 for non-functional).
- `nodeIndex` (integer) Car index if 0 is nearest to camera, 1 is second nearest and so on.
- `visibleIndex` (integer) Car index if 0 is nearest to camera and visible in main camera, 1 is second nearest visible and so on. Cars outside of main camera would have 255 here.
- `activeLOD` (integer) 0-based index of visible LOD.
- `distanceToCamera` (number) Distance to camera in meters.
- `splinePosition` (number) Position of car along the track, 0 for starting line, 1 for finishing line.
- `driftPoints` (number) Drift points (calculated in any racing mode).
- `driftInstantPoints` (number) Drift instant points (calculated in any racing mode).
- `driftComboCounter` (integer) Drift combo counter (calculated in any racing mode).
- `collisionDepth` (number) How deep is current collision, in meters (generally it’s better to use change of car speed to estimate collision intensity though, depth is much less predictable).
- `collisionPosition` (vec3) Coordinates of current collision in car space.
- `collidedWith` (integer) 0 for track, non-zero for cars.
- `lapCutsCount` (integer) Number of lap cuts in current lap. Physics-only (see `ac.CarState.physicsAvailable`)
- `lastLapCutsCount` (integer) Number of lap cuts in last lap.
- `aiLevel` (number) AI level from 0 to 1 (or -1 if there is no AI).
- `aiAggression` (number) AI aggression from 0 to 1 (or -1 if there is no AI). Note: aggression level set in a launcher will be multiplied by 0.95, so 100% aggression will show up as 0.95 here.
- `aiRainTyres` (number) `True` if currently AI is using rain tyres.
- `currentSplits` (integer[]) Time for different splits of current lap, in milliseconds. Items start with 0. To get number of elements, use `#state.currentSplits`.
- `lastSplits` (integer[]) Time for different splits of last lap, in milliseconds. Items start with 0. To get number of elements, use `#state.lastSplits`.
- `bestSplits` (integer[]) Best splits times (and not splits of best lap), in milliseconds, unlike other split lists, this one is populated with zeroes for sectors without time from the start. Items start with 0. To get number of elements, use `#state.bestSplits`.
- `bestLapSplits` (integer[]) Splits times of best lap (not necessarily best split times in itself), in milliseconds. Items start with 0. To get number of elements, use `#state.bestLapSplits`.
- `wheels` (ac.StateWheel[]) 4 items, starts with 0.
- `extrapolatedMovement` (boolean) If set to false and you’re adding objects moving close to cars, use ac.CarState.timestamp to estimate dt in such a way that it would match car physics time
- `isActive` (boolean) True if car is currently active (changes to `false` for disconnected cars if server does not have them visible).
- `isConnected` (boolean) True if car is currently connected (cars can be disconnected online), or if car is not a remote one.
- `isRemote` (boolean) True if car is controlled by another player online.
- `isAIControlled` (boolean) True if car is controlled by AI (or that autopilot thing).
- `isUserControlled` (boolean) True if car is controlled by user (controls could be disabled completely).
- `isLapValid` (boolean) True if current lap is valid. Physics-only (see `ac.CarState.physicsAvailable`)
- `isLastLapValid` (boolean) True if last lap is valid.
- `isCameraOnBoard` (boolean) True if camera is inside this car.
- `isInPitlane` (boolean) True if car is in pits area.
- `isInPit` (boolean) True if car is parked in its pit stop place.
- `isRetired` (boolean) 
- `isEngineLimiterOn` (boolean) 
- `isGearGrinding` (boolean) 
- `hasTurningLights` (boolean) There could be other components linked to turn signals state, but this variable is about emissives tied to turn signals.
- `headlightsActive` (boolean) 
- `brakeLightsActive` (boolean) 
- `flashingLightsActive` (boolean) 
- `hornActive` (boolean) 
- `focused` (boolean) 
- `focusedOnInterior` (boolean) 
- `useInternalAudio` (boolean) If `true`, `…_int` variants of audio events are used.
- `isHidingLabels` (boolean) If you’re drawing a map, don’t show cars with this flag on (those would be disconnected cars or, for example, cars acting like traffic). Flag can change during the race
- `isDriftBonusOn` (boolean) Drift bonus flag (calculated in any racing mode).
- `isDriftValid` (boolean) Is drift valid (calculated in any racing mode).
- `isRaceFinished` (boolean) Car has finished the race. Since 0.2.8, non-race sessions are never marked as finished.
- `isDriverVisible` (boolean) Driver can be hidden by apps or Lua scripts too.
- `isDriverDoorOpen` (boolean) 
- `isDriverSeatbeltOn` (boolean) Same as `SEATBELT` input.
- `hazardLights` (boolean) 
- `turningLeftLights` (boolean) 
- `turningRightLights` (boolean) 
- `turningLeftOnly` (boolean) 
- `turningRightOnly` (boolean) 
- `lowBeams` (boolean) `true` if low beams switch is currently active (doesn’t depend on the state of headlights, just on the switch state).
- `highBeams` (boolean) `true` if high beams are currently active (either with headlights active, or with high beams flashing).
- `extraA` (boolean) 
- `extraB` (boolean) 
- `extraC` (boolean) 
- `extraD` (boolean) 
- `extraE` (boolean) 
- `extraF` (boolean) 
- `extraG` (boolean) 
- `extraH` (boolean) 
- `extraI` (boolean) 
- `extraJ` (boolean) 
- `extraK` (boolean) 
- `extraL` (boolean) 
- `extraM` (boolean) 
- `extraN` (boolean) 
- `extraO` (boolean) 
- `extraP` (boolean) 
- `extraQ` (boolean) 
- `extraR` (boolean) 
- `extraS` (boolean) 
- `extraT` (boolean) 
- `kersCharging` (boolean) 
- `turningLightsActivePhase` (boolean) 
- `anyFlamesActive` (boolean) `true` if any of car flames are currently shown.
- `justJumped` (boolean) Changes to `true` for a single frame when car jumps.
- `resetCounter` (integer) Increases by 1 each time a car resets.
- `kersCharge` (number) 
- `kersInput` (number) 
- `kersCurrentKJ` (number) 
- `kersMaxKJ` (number) 
- `kersLoad` (number) 
- `distanceDrivenTotalKm` (number) 
- `distanceDrivenSessionKm` (number) 
- `poweredWheelsSpeed` (number) Can be used for speedometers if they’re not using GPS and instead use powered wheels, value in km/h.
- `batteryVoltage` (number) 
- `oilPressure` (number) 
- `oilTemperature` (number) 
- `exhaustTemperature` (number) 
- `wiperModes` (integer) Number of wiper modes, no less than 1 (wipers disabled state counts like a 0th mode, all cars would have that).
- `wiperMode` (integer) Stores previous active mode when wipers are disabled, but still have a loop to complete.
- `wiperSelectedMode` (integer) Actual selected mode.
- `wiperSpeed` (number) 
- `wiperProgress` (number) 
- `bodyWetness` (number) How wet is car exterior, approximation from 0 to 1 (actual wetness is in 2D map).
- `compass` (number) Angle of where car is heading, from 0 to 360 (0/360 for north, 90 for east, etc.)
- `lapTimeMs` (integer) Time of current lap in milliseconds.
- `bestLapTimeMs` (integer) Time of best lap of this session in milliseconds.
- `previousLapTimeMs` (integer) Time of last lap in milliseconds.
- `lapCount` (integer) Number of completed laps in this session (within connection online), including spoiled laps.
- `currentSector` (integer) 0-based index of current track split.
- `previousSectorTime` (integer) Time of previous split in milliseconds, or 0 if it’s a first split.
- `racePosition` (integer) Position of a car in the race, 1 for first, 2 for second, etc.
- `drivenInRace` (number) Distance driven in current race in meters (can be used for estimating race position, based on number of completed laps and current lap progress, in practice or qualify session resets each lap).
- `estimatedLapTimeMs` (integer) Based on best lap and performance meter (delta of this lap time vs best lap time).
- `performanceMeter` (number) Performance meter comparing this lap with best, seconds.
- `performanceMeterSpeedDifferenceMs` (number) In AC performance app, there is that red/green bar, it shows this value.
- `sessionLapCount` (integer) 
- `compoundIndex` (integer) Index of currently selected tyre compounds.
- `sessionID` (integer) Index of a car in an online race (differs from regular car index: sessionID is an index in entry list, but car.index of your car is always zero)
- `physicsAvailable` (boolean) Cars in replays, or remote cars online do not have regular physics component running, so some data will be missing. Such fields are marked as physics-only in comments.
- `speedLimiterInAction` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `absInAction` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `tractionControlInAction` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `hasUserBrakeBias` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `hasCockpitMGUHMode` (boolean) 
- `hasCockpitERSDelivery` (boolean) 
- `hasCockpitERSRecovery` (boolean) 
- `drsPresent` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `drsAvailable` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `drsActive` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `kersPresent` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `kersHasButtonOverride` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `kersButtonPressed` (boolean) Physics-only (see `ac.CarState.physicsAvailable`)
- `mguhChargingBatteries` (boolean) 
- `manualPitsSpeedLimiterEnabled` (boolean) Returns `true` if manual pits speed limiter is currently active. Physics-only (see `ac.CarState.physicsAvailable`)
- `userSpeedLimiterEnabled` (boolean) Returns `true` if custom physics speed limiter is currently active. Not the same as pit limiter (or manually operated speed limiter). Physics-only (see `ac.CarState.physicsAvailable`)
- `autoShift` (boolean) Returns `true` if automatic shifting is active. Physics-only (see `ac.CarState.physicsAvailable`)
- `autoClutch` (boolean) Returns `true` if auto-clutch is active. Physics-only (see `ac.CarState.physicsAvailable`)
- `mgukDelivery` (integer) Starts with 0.
- `mgukDeliveryCount` (integer) 
- `mgukRecovery` (integer) From 0 to 10 (for 100%).
- `tractionControlMode` (integer) 0 for disabled TC. Physics-only (see `ac.CarState.physicsAvailable`)
- `absMode` (integer) 0 for disabled ABS. Physics-only (see `ac.CarState.physicsAvailable`)
- `engineBrakeSettingsCount` (integer) 
- `currentEngineBrakeSetting` (integer) 
- `fuelPerLap` (number) Uses original AC fuel estimation. Zero until value is available. Physics-only (see `ac.CarState.physicsAvailable`)
- `differentialPreload` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `differentialCoast` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `differentialPower` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `awdFrontShare` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `awdCenterLock` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `drivetrainTorque` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `drivetrainPower` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `brakeBias` (number) Physics-only (see `ac.CarState.physicsAvailable`)
- `speedLimiter` (number) Returns pit limiter speed in km/h (80 by default unless session specified different settings) or 0 if no limit is currently active. Physics-only (see `ac.CarState.physicsAvailable`)
- `turboBoosts` (number[]) Values per each turbo, up to 8 (if there are less turbos in a car, rest are zeroes). Physics-only (see `ac.CarState.physicsAvailable`) 8 items, starts with 0.
- `turboWastegates` (number[]) Values per each turbo, up to 8 (if there are less turbos in a car, rest are zeroes). Physics-only (see `ac.CarState.physicsAvailable`) 8 items, starts with 0.
- `tractionControl2` (number) 
- `fuelMap` (integer) Current fuel map preset.
- `steerTorque` (number) 
- `ffbSteerAssist` (number) Value of `STEER_ASSIST` from car.ini. Physics-only (see `ac.CarState.physicsAvailable`)
- `ffbBase` (number) Value of `FFMULT` from car.ini. Physics-only (see `ac.CarState.physicsAvailable`)
- `ffbFinal` (number) 
- `ffbPure` (number) 
- `ffbMultiplier` (number) For 100% FFB multiplier, this value is set to 1.
- `aeroLiftFront` (number) Aero lift coefficient in front.
- `aeroLiftRear` (number) Aero lift coefficient in rear.
- `aeroDrag` (number) Aero drag coefficient.
- `caster` (number) Caster angle in degrees.
- `rideHeight` (number[]) 0 for front, 1 for rear. 2 items, starts with 0.
- `p2pStatus` (integer) 
- `p2pActivations` (integer) 
- `altitude` (number) Altitude in meters above sea level.
- `ambientOcclusion` (number) Ambient occlusion value computed from prebaked data from track’s VAO patch. 0 for car fully shadowed (in a tunnel), 1 for car outside. Affected by WeatherFX adjustments.
- `rawAmbientOcclusion` (number) Ambient occlusion value computed from prebaked data from track’s VAO patch. 0 for car fully shadowed (in a tunnel), 1 for car outside. Not affected by WeatherFX adjustments.
- `carCamerasCount` (integer) Number of F6 cameras.
- `currentPenaltyType` (ac.PenaltyType) Current penalty type (set only for user car).
- `currentPenaltyParameter` (integer) Parameter of current penalty (role depends on penalty type).
- `extraIconsCount` (integer) Extra driver icons set with `ui.UserIconsLayer()`. To draw, use `carN::special::driver::X` texture name.

</details>

### `ac.getCarPhysics(i)` — fizik durumu (sadece oyuncu aracı) (16 alan)

`isAvailable`, `gearRatio`, `finalRatio`, `airDensity`, `awd2MaxTorque`, `awd2CurrentLockTorque`, `awd2Ramp`, `engineInertia`, `drivetrainInertia`, `drivetrainVelocity`, `clutchInertia`, `clutchState`, `wheels`, `scriptControllerInputs`, `gearRatios`, `wings`

<details><summary>Açıklamalar</summary>

- `isAvailable` (boolean) 
- `gearRatio` (number) 
- `finalRatio` (number) 
- `airDensity` (number) Air density affected by slip stream.
- `awd2MaxTorque` (number) 
- `awd2CurrentLockTorque` (number) 
- `awd2Ramp` (number) 
- `engineInertia` (number) 
- `drivetrainInertia` (number) 
- `drivetrainVelocity` (number) 
- `clutchInertia` (number) 
- `clutchState` (number) 
- `wheels` (ac.StateWheelPhysics[]) 4 items, starts with 0.
- `scriptControllerInputs` (number[]) 256 items, starts with 0.
- `gearRatios` (number[]) Items start with 0. To get number of elements, use `#state.gearRatios`.
- `wings` (ac.StateWingPhysics[]) Items start with 0. To get number of elements, use `#state.wings`.

</details>

### `car.wheels[i]` — tekerlek (62 alan)

`tyreRadius`, `tyreWidth`, `rimRadius`, `tyreDirty`, `tyreWear`, `tyreVirtualKM`, `tyreGrain`, `tyreBlister`, `tyreFlatSpot`, `tyreStaticPressure`, `tyrePressure`, `temperatureMultiplier`, `tyreCoreTemperature`, `tyreInsideTemperature`, `tyreMiddleTemperature`, `tyreOutsideTemperature`, `tyreOptimumTemperature`, `brakeTemperature`, `discTemperature`, `angularSpeed`, `slip`, `slipAngle`, `slipRatio`, `ndSlip`, `load`, `loadK`, `speedDifference`, `camber`, `toeIn`, `suspensionDamage`, `suspensionTravel`, `tyreLoadedRadius`, `waterThickness`, `dx`, `dy`, `mz`, `fx`, `fy`, `contactNormal`, `contactPoint`, `position`, `look`, `up`, `outside`, `velocity`, `transform`, `transformWheel`, `surfaceDirt`, `surfaceSectorID`, `surfaceSectionIndex`, `surfaceGrip`, `surfaceDamping`, `surfaceGranularity`, `surfaceVibrationGain`, `surfaceVibrationLength`, `surfacePitlane`, `surfaceValidTrack`, `isBlown`, `isHot`, `isSpecialSurface`, `surfaceType`, `surfaceExtendedType`

<details><summary>Açıklamalar</summary>

- `tyreRadius` (number) Tyre radius in meters.
- `tyreWidth` (number) Tyre width in meters.
- `rimRadius` (number) Rim radius in meters (older cars might not have it).
- `tyreDirty` (number) Dirt levels for tyres, four values, from 0 to 1.
- `tyreWear` (number) Tyre wear, from 0 to 1.
- `tyreVirtualKM` (number) Driven distance for each tyre in km (resets when tyres change, not an actual distance, rate of change depends on wear multiplier).
- `tyreGrain` (number) 
- `tyreBlister` (number) 
- `tyreFlatSpot` (number) 
- `tyreStaticPressure` (number) Static tyre pressure.
- `tyrePressure` (number) Dynamic tyre pressure.
- `temperatureMultiplier` (number) 
- `tyreCoreTemperature` (number) Core tyre temperature, in °C.
- `tyreInsideTemperature` (number) Inside tyre temperature, in °C.
- `tyreMiddleTemperature` (number) Middle tyre temperature, in °C.
- `tyreOutsideTemperature` (number) Outside tyre temperature, in °C.
- `tyreOptimumTemperature` (number) Optimum tyre temperature, in °C.
- `brakeTemperature` (number) Brakes temperature, in °C, or 0 if it’s not available (for most cars).
- `discTemperature` (number) 
- `angularSpeed` (number) 
- `slip` (number) 
- `slipAngle` (number) Angle between the desired direction and the actual direction of the vehicle, in degrees.
- `slipRatio` (number) 
- `ndSlip` (number) 
- `load` (number) Tyre load in N (warning: does not have correct values for remote cars online or in replays, use `.loadK` if you want a rough but stable estimation instead)
- `loadK` (number) 
- `speedDifference` (number) Difference in speed between tyre and ground surface in contact point.
- `camber` (number) Camber angle in degrees.
- `toeIn` (number) Toe angle in degrees.
- `suspensionDamage` (number) 
- `suspensionTravel` (number) 
- `tyreLoadedRadius` (number) 
- `waterThickness` (number) Water thickness in meters.
- `dx` (number) 
- `dy` (number) 
- `mz` (number) Self aligning torque.
- `fx` (number) 
- `fy` (number) 
- `contactNormal` (vec3) Normal of a contact point surface.
- `contactPoint` (vec3) Contact point position in world coordinates.
- `position` (vec3) Wheel position in world coordinates.
- `look` (vec3) Wheel (SUSP_…) heading vector.
- `up` (vec3) Wheel (SUSP_…) vector directed updwards.
- `outside` (vec3) Points outside of car (to the left of tyre for left tyres, to the right for right).
- `velocity` (vec3) 
- `transform` (mat4x4) Wheel (SUSP_…) transformation.
- `transformWheel` (mat4x4) Wheel (WHEEL_…) transformation.
- `surfaceDirt` (number) Dirt additive coefficient of the surface below the wheel.
- `surfaceSectorID` (integer) Sector ID of the surface below the wheel.
- `surfaceSectionIndex` (integer) Index of section in “surfaces.ini” corresponding with surface the wheel is on, or -1 if it’s not available
- `surfaceGrip` (number) Grip of the surface below the wheel.
- `surfaceDamping` (number) Damping of the surface below the wheel.
- `surfaceGranularity` (number) Damping of the surface below the wheel.
- `surfaceVibrationGain` (number) Vibration gain of the surface below the wheel.
- `surfaceVibrationLength` (number) Vibration length of the surface below the wheel.
- `surfacePitlane` (boolean) If surface below the wheel is in pitlane.
- `surfaceValidTrack` (boolean) If surface below the wheel is valid track.
- `isBlown` (boolean) 
- `isHot` (boolean) 
- `isSpecialSurface` (boolean) Do not play skids audio if this value is `true`.
- `surfaceType` (ac.SurfaceType) Basic type hint used for visuals.
- `surfaceExtendedType` (ac.SurfaceExtendedType) Extended type hint, more types will be added later.

</details>

### `getCarPhysics(i).wheels[i]` — tekerlek fiziği (10 alan)

`brakeTorque`, `handbrakeTorque`, `electricTorque`, `feedbackTorque`, `angularInertia`, `shaftVelocity`, `shaftInertia`, `tyreCarcassTemperature`, `roadTemperature`, `discCoreTemperature`

<details><summary>Açıklamalar</summary>

- `brakeTorque` (number) 
- `handbrakeTorque` (number) 
- `electricTorque` (number) 
- `feedbackTorque` (number) 
- `angularInertia` (number) 
- `shaftVelocity` (number) 
- `shaftInertia` (number) 
- `tyreCarcassTemperature` (number) Only available with custom physics tyres.
- `roadTemperature` (number) Can vary from AC road temperature in shadows if option is enabled.
- `discCoreTemperature` (number) Can vary from AC road temperature in shadows if option is enabled.

</details>

### `ac.getSim()` — simülasyon durumu (164 alan)

`windowWidth`, `windowHeight`, `msaaSamples`, `worldDetailLevel`, `windowSize`, `isFullscreen`, `isVSyncActive`, `isPostProcessingActive`, `isVRMode`, `isOculusMode`, `isOpenVRMode`, `isTripleMode`, `isCustomVideoMode`, `isVRConnected`, `staticReflections`, `directMessagingAvailable`, `directUDPMessagingAvailable`, `allowTyreBlankets`, `isReplayOnlyMode`, `isOnlineRace`, `isShowroomMode`, `isPreviewsGenerationMode`, `freeCameraAllowed`, `isPitsSpeedLimiterForced`, `customAISplinesAllowed`, `isFSRActive`, `isTripleFSRActive`, `isLinearColorSpaceActive`, `allowedTyresOut`, `resultScreenTime`, `raceOverTime`, `mechanicalDamageRate`, `tyreConsumptionRate`, `fuelConsumptionRate`, `gravity`, `inputMode`, `windowHandle`, `currentVAOMode`, `neutralSunTrajectoryTimestamp`, `isMakingScreenshot`, `time`, `gameTime`, `systemTime`, `frame`, `cameraPosition`, `cameraPositionDelta`, `cameraLook`, `cameraUp`, `cameraSide`, `cameraFOV`, `cameraClipNear`, `cameraClipFar`, `cameraDOFFactor`, `cameraDOFFocus`, `cameraExposure`, `exposureMultiplier`, `carsCount`, `closelyFocusedCar`, `focusedCar`, `dt`, `isPaused`, `isInMainMenu`, `isLookingAtSessionResults`, `needsDPad`, `isLive`, `isFreeCameraOutside`, `isReplayActive`, `isFocusedOnInterior`, `isTripleScreenAvailable`, `controlsWithShifter`, `isVirtualMirrorActive`, `hideSteeringWheel`, `hideArmsInCockpit`, `isVirtualMirrorForced`, `isWindowForeground`, `isWeatherFXActive`, `isVAOPatchLoaded`, `orbitOnboardCamera`, `cameraJumped`, `isMouseSteeringActive`, `ambientLightingMultiplier`, `ambientTemperature`, `roadTemperature`, `lightSuggestion`, `roadGrip`, `windSpeedKmh`, `windVelocityKmh`, `windDirectionDeg`, `rainIntensity`, `rainWetness`, `rainWater`, `bestLapTimeMs`, `timeTotalSeconds`, `timeHours`, `timeMinutes`, `timeSeconds`, `timeMultiplier`, `timestamp`, `dayOfYear`, `raceFlagType`, `raceFlagCause`, `isSessionStarted`, `isSessionFinished`, `raceSessionType`, `timeToSessionStart`, `sessionTimeLeft`, `currentSessionIndex`, `sessionsCount`, `leaderLapCount`, `trackLengthM`, `speedLimitKmh`, `fps`, `fpsCapped`, `physicsLate`, `cpuOccupancy`, `cpuTime`, `firstPersonCameraFOV`, `cameraMode`, `driveableCameraMode`, `carCameraIndex`, `trackCamerasSetsCount`, `trackCamerasSet`, `audioMasterVolume`, `whiteReferencePoint`, `specialEvent`, `isTimedRace`, `penaltiesEnabled`, `fixedSetup`, `leaderLastLap`, `timeRaceEnded`, `timeRaceLastLap`, `timeRaceAdditionalLap`, `idealLineShown`, `damageDisplayerShown`, `driverNamesShown`, `isAdmin`, `sentAdminCommand`, `pitsSpeedLimit`, `pitWindowStartTime`, `pitWindowEndTime`, `replayFrameMs`, `timeSinceLastUDPPacket`, `replayFrames`, `replayCurrentFrame`, `replayPlaybackRate`, `weatherType`, `weatherSkyOcclusion`, `baseAltitude`, `currentQuickPitPreset`, `connectedCars`, `randomSeed`, `equinoxSunTrajectory`, `explicitDateSet`, `lapSplits`, `handshakeServerTime`, `currentServerTime`, `currentSessionTime`, `skyColor`, `horizonColor`, `fogColor`, `lightColor`, `lightDirection`, `originShift`, `weatherConditions`


## 5. `ac.*` fonksiyonları (577 adet, konuya göre)

### Kamera / görüntü (53)

- `ac.getCameraPosition()`
- `ac.getCameraUp()`
- `ac.getCameraSide()`
- `ac.getCameraForward()`
- `ac.getCameraDirection()` — This vector is pointing backwards! Only kept for compatibility. For proper one, use `ac.getCameraForward()`.
- `ac.getCameraFOV()` — Value in degrees.
- `ac.getCameraPositionTo(r)`
- `ac.getCameraUpTo(r)`
- `ac.getCameraSideTo(r)`
- `ac.getCameraForwardTo(r)`
- `ac.getCameraDirectionTo(r)`
- `ac.getCameraPositionRelativeToCar()` — Returns camera position in car coordinates system.
- `ac.isVisibleInMainCamera(pos, radius, gSpace, includeFarPlane)` — Can be called from anywhere. Checks if given point is within main camera frustum (or several, in triple screen mode) or not. For checking if something is within current c
- `ac.onScreenshot(callback)` — Sets a callback which will be called when a new screenshot is made.
- `ac.setAppWindowVisible(appID, windowFilter, visible)` — Shows a hidden app window or few while current script is alive (temporarily removing “HIDDEN” window flag).
- `ac.setCurrentCamera(mode)` — Sets current camera mode.
- `ac.setCurrentDrivableCamera(mode)` — Sets current drivable camera mode.
- `ac.setCurrentCarCamera(mode)` — Sets current car camera index.
- `ac.setCurrentTrackCamera(set)` — Sets current track cameras set.
- `ac.focusCar(index)` — Focus on a certain car.
- `ac.setCameraPosition(pos)` — Moves free camera to a certain position. Note: if you want to control camera fully, consider grabbing camera instead.
- `ac.setCameraDirection(look, up)` — Orients free camera. Note: if you want to control camera fully, consider grabbing camera instead.
- `ac.setCameraFOV(fov)` — Sets FOV of a free camera. Note: if you want to control camera fully, consider grabbing camera instead.
- `ac.setCameraExposure(exposure)` — Sets exposure of a free camera. Note: if you want to control camera fully, consider grabbing camera instead.
- `ac.setCameraDOF(distance)` — Sets DOF distance of a free camera. Note: if you want to control camera fully, consider grabbing camera instead.
- `ac.setAppsHidden(hidden)` — Hides or shows any apps.
- `ac.setOrbitOnboardCamera(orbit)`
- `ac.setCameraCPLSettings(intensity, rotation, offset, iridescenceBoost)` — Override camera CPL settings.
- `ac.overrideCameraClipPlanes(nearPlane, farPlane)` — Override camera clip planes. Pass distances in meters, or `nil` to disable the override.
- `ac.setOnboardCameraFOV(fovDeg)` — Use `ac.setFirstPersonCameraFOV()`
- `ac.resetOnboardCameraFOV()` — Use `ac.resetFirstPersonCameraFOV()`
- `ac.setFirstPersonCameraFOV(fovDeg)` — Changes FOV for first person camera. To get current value, use `ac.getSim().firstPersonFOV`.
- `ac.resetFirstPersonCameraFOV()` — Resets FOV for first person camera to 56° (which is a default, but actual value is loaded from “assettocorsa/cfg/camera_onboard.ini”). To get current value, use `ac.getSi
- `ac.getOnboardCameraParams(carIndex)` — Gets onboard camera parameters for a specific car.
- `ac.getOnboardCameraDefaultParams(carIndex)` — Gets default onboard camera parameters for a specific car (loading them from “car.ini” from car data).
- `ac.setOnboardCameraParams(carIndex, params, save)` — Sets onboard camera parameters for a specific car. Optional saving works for the first car only.
- `ac.areOnboardCameraParamsNeedSaving()` — Returns `true` if current camera parameters need saving (when default camera positioning app shows a “Save needed” label).
- `ac.makeScreenshot(filename, format, callback)` — Runs asyncronously.
- `ac.makeSimpleScreenshot(format, callback)` — Returns a binary image JPG data containing stuff on the screen (with HUD and everything). Asynchronous version.
- `ac.isWindowOpen(windowID)` — Returns `true` if window with given ID is opened.
- `ac.isWindowCollapsed(windowID)` — Returns `true` if window with given ID is in collapse state.
- `ac.setWindowSizeConstraints(windowID, minSize, maxSize)` — Changes window size constraints, can be used to update window size live.
- `ac.isWindowPinned(windowID)` — Returns `true` if window with given ID is pinned.
- `ac.setWindowOpen(windowID, opened)` — Opens or closes a window.
- `ac.setWindowIcon(windowID, iconID)` — Changes window icon. If your icon doesn’t change every frame, use this function instead of a live icon.
- `ac.setWindowTitle(windowID, title)` — Sets window title.
- `ac.setWindowBackground(windowID, color, darkHeader)` — Sets window background color.
- `ac.setWindowNotificationCounter(windowID, counter)` — Adds notifications counter to a window for something like amount of unread messages. Make sure to set it back to 0 when things have been seen.
- `ac.windowFading()` — If current window fades (has “FADING” flag), this value would store fading value. If background is fully opaque, value is 0.
- `ac.grabCamera(reason)` — Tries to grabs camera for script to control it. If successfull, returns instance of `ac.GrabbedCamera` which you can then use to move camera, rotate it and alter some of 
- `ac.getAppWindows()` — Collect information about available windows.
- `ac.accessAppWindow(windowName)` — Looks for a certain window of either an original AC app, a Python or a Lua app. Use `ac.getAppWindows()` to get a list of available apps. If found, this wrapper can be us
- `ac.accessCarCamera(cameraIndex)`

### Hava / ışık / gökyüzü (37)

- `ac.getTrackTimezoneBaseDst(time)` — Returns timezone offset for the track in seconds.
- `ac.lapTimeToString(time, allowHours)` — Turns time in milliseconds into common lap time presentation, like 00:00.123. If minutes exceed 60, hours will also be added, but only if `allow_hours` is `true` (default
- `ac.getSunAngle()` — Value in degrees.
- `ac.getSunPitchAngle()` — Value in degrees.
- `ac.getSunHeadingAngle()` — Value in degrees.
- `ac.getConditionsTimeScale()` — Returns current time multiplier.
- `ac.getWindVelocity()` — Value is in m/s.
- `ac.getWindVelocityTo(r)` — Value is in m/s.
- `ac.isWeatherFxActive()`
- `ac.getCarRealTimeLeaderboardPosition(carIndex)` — Returns real time car position, same as Python function with the same name. Does not work online. For an alternative solution, get position calculated by CSP via `ac.getC
- `ac.getSkyFeatureDirection(skyFeature, distance, time)` — Get direction to a sky feature in world-space (corrected for track heading). If feature is not available, returns a zero vector.
- `ac.getSkyStarDirection(declRad, rightAscRad)` — Get direction to a star in the sky in world-space (corrected for track heading). If feature is not available, returns a zero vector.
- `ac.getPerformanceCPUAndGPUTime()`
- `ac.getTimeZoneOffset()`
- `ac.getTimeZoneDstOffset()`
- `ac.getTimeZoneBaseOffset()`
- `ac.getTrackDateTime()` — Returns floading point number of seconds since 1970/01/01 that can be used for driving track animations in such a way that if time multiplier is set to 0 or above 1, thin
- `ac.evaluateLapTime()` — Estimates lap time and sector times for main car using AC function originally used by Time Attack mode. Could be helpful in creating custom time attack modes. Uses “ideal
- `ac.TemperatureParams(ambient, road)`
- `ac.WindParams(direction, speedFrom, speedTo)`
- `ac.setNeutralSunTrajectoryTimestamp(value)`
- `ac.forceSnow()`
- `ac.highlightCar(carIndex, highlightColor)` — Add fake light to a car (not realistic at all, can be used for something like bringing an attention to a recently teleported car with its collisions disabled).  When scri
- `ac.debugLights(filter, count, mode, distance)` — Enables and disables lights debugging outlines. Parameters can be used to tune which lights would generate those outlines.
- `ac.debugRainRacingLine(active)` — Enables and disables racing line debug view. Racing line is used to keep track of dynamic wetness offset caused by cars drying up the track when driving by.
- `ac.setWeatherExactUTC0Timestamp(time, explicitDate)` — Replace current time entirely. Also alters timezone if new time would have different DST. Note: `time` here is a global timestamp with no timezone correction! Can’t be re
- `ac.setWeatherTimeMultiplier(multiplier)` — Sets current time multiplier used by WeatherFX. Affects offline races only.
- `ac.setWeatherTimeOffset(offset, instant)` — Adds or subtracts time offset, useful for debugging. Affects offline races only. Offset above 24 hours counts as race date being set, so if it wasn’t set in session setti
- `ac.setTrackTimezoneOffset(offset)` — Sets timezone offset for the track in seconds. Not available online. Be careful: unlike with configs, this method does not have any coordinate-related sanity checks.
- `ac.setHeadlights(active)`
- `ac.flashLights()` — Activates flashing lights for a bit.
- `ac.setDaytimeLights(active)`
- `ac.setBrakingLightsThreshold(threshold)`
- `ac.setTurningLights(mode)`
- `ac.timeToNextVote(voteType)` — If `ac.canCastVote()` returns `false` due to cooldown, this option will return time in seconds until the next vote will become available.
- `ac.exportLapTimesToCSV(destination)` — Asks AC to do the thing it does when “Print” button in original menu was pressed, saving lap times in a CSV file.
- `ac.LightSource(lightType)` — Light source on the scene. Starts working immediately after creation. Use `:dispose()` to remove it.

### Pist / yüzey / konum (56)

- `ac.hasTrackSpline()` — Refers to AI spline.
- `ac.worldCoordinateToTrackProgress(v)` — Finds nearest point on track AI spline (fast_lane) and returns its normalized position. If there is no track spline, returns -1.
- `ac.getTrackAISplineSides(v)` — Returns distance from AI spline to left and right track boundaries.
- `ac.trackProgressToWorldCoordinate(v, linear)`
- `ac.trackProgressToWorldCoordinateTo(v, r, linear)`
- `ac.worldCoordinateToTrack(v)` — Converts world coordinates into track coordinates. Track coordinates:  - X for normalized position (0 — right in the middle, -1 — left side of the track, 1 — right size);
- `ac.trackCoordinateToWorld(v)` — Converts track coordinates into world coordinates. Track coordinates:  - X for normalized position (0 — right in the middle, -1 — left side of the track, 1 — right size);
- `ac.getTrackCoordinatesDeg(worldPos)` — Returns track world coordinates in degrees.
- `ac.getTrackSectorName(trackProgress)` — Name of a sector.
- `ac.getTrackId()` — Use `ac.getTrackID()` instead.
- `ac.getTrackID()` — Returns track ID (name of its folder).
- `ac.getTrackLayout()` — Returns track layout ID (name of layout folder, without name of track folder), or empty string if there is no layout.
- `ac.getTrackFullID(separator)` — Returns full track ID (name of track folder and layout folder joined by some string, or just name of track folder if there is no layout).
- `ac.getTrackName()` — Returns track name (as set in its JSON file).
- `ac.getTrackDataFilename(fileName)` — Given name, returns a path like …/assettocorsa/content/tracks/[track ID]/data/[name], taking into account layout as well.
- `ac.getTrackUpcomingTurn(carIndex)` — Distance and turn angle (in degrees) for the upcoming turn. If failed to compute, both would be -1. If car is facing wrong way, turn angle is either 180° or -180° dependi
- `ac.forceVisibleHeadNodes(carIndex, force)` — Forces driver head to be visible even with cockpit camera.
- `ac.onTrackPointCrossed(carIndex, progress, callback)` — Sets a callback which will be called when car crosses a certain point on a track. Won’t be triggered if car is in pitlane or recently jumped. Time is relative to the same
- `ac.mediaNextTrack()` — Switches to the next track in currently active music player (by simulating media key press).
- `ac.mediaPreviousTrack()` — Switches to the previous track in currently active music player (by simulating media key press).
- `ac.getRealTrackHeadingAngle()`
- `ac.getTrackIR()` — Returns reference to a structure with TrackIR-related values. Very cheap to use.
- `ac.INIConfig.trackData(fileName)` — Load track data INI file. Can be used by track scripts which might not always  have access to those files directly. Returned file can’t be saved.
- `ac.INIConfig.trackConfig()` — Returns CSP config for a track. Might be slow: some of those configs are huge. Make sure to cache the resulting value if you need to reuse it. Returned file can’t be save
- `ac.TrackConditions(sessionStart, sessionTransfer, randomness, lapGain)` — State of the track surface.
- `ac.getTrackConfig(section, key, defaultValue)` — Reads a value from the config of currently loaded track. To use it, you need to specify `defaultValue` value, it would be used to determine the type of the value you need
- `ac.MeshVertex(pos, normal, uv)` — Mesh vertex.
- `ac.MeshVertex.new(pos, normal, uv)` — Creates new mesh vertex.
- `ac.emptySceneReference()`
- `ac.findNodes(s)` — Creates a new scene reference containing nodes collected with a filter from root node associated with current script. For most scripts it would be an AC root node. For tr
- `ac.findMeshes(s)` — Creates a new scene reference containing meshes collected with a filter from root node associated with current script. For most scripts it would be an AC root node. For t
- `ac.findSkinnedMeshes(s)` — Creates a new scene reference containing skinned meshes collected with a filter from root node associated with current script. For most scripts it would be an AC root nod
- `ac.onTrackPhysicsObjectCarCollision(name, carIndex, callback)` — Adds a listener that will be called once a car touches an track physics object for first time, or after touching another object. State resets once object is reset (with n
- `ac.setExtraTrackLODMultiplier(multiplier)` — Might be helpful if you’re moving camera really far away and not grabbing it for some reason. Nice Screenshots module sets it to 10 when starting to make a screenshot.
- `ac.overrideTrackTrees(listName, data, alignSurface)` — Replace track trees with a custom list. Use `listKey` not set in “ext_config.ini” to add new lists. Feel free to call this function each frame if amount of data is small.
- `ac.configureSceneTweaks()` — Override scene tweaks.
- `ac.setTrackCoordinatesDeg(coords, headingDeg)` — Sets track world coordinates in degrees. Not available online.
- `ac.isAISplineRecorderPresent()` — Returns `true` if AI spline recorder is present and ready to do some recording.
- `ac.isAISplineRecorderActive()`
- `ac.isAISplineRecorderRecordingPitLane()`
- `ac.getAISplineRecordedFrames()`
- `ac.startAISplineRecorder(recordPitLane)`
- `ac.stopAISplineRecorder(saveResult, callback)`
- `ac.setTrackCondition(key, value)` — Sets custom track condition input.
- `ac.setTrackConditionInput(conditionInput, value)` — Use `ac.setTrackCondition()` instead.
- `ac.SimpleMesh.carShape(carIndex, includeDriver)` — Creates a description of a simple mesh with a car shape. Usually is generated from LOD D if present and not too large, otherwise uses collider mesh.
- `ac.SimpleMesh.carCollider(carIndex, actualCollider)` — Creates a description of a simple mesh with a car collider.
- `ac.SimpleMesh.trackLine(lineType, absolute, relative)` — Creates a description of a simple mesh with a track line.
- `ac.TrackPaint()`
- `ac.SceneTweaks()` — Driver seat parameters.
- `ac.TrackCondition(expression, offset, defaultValue)` — Creates a wrapper to access track condition. If you want to get the value often, consider caching and reusing the wrapper.
- `ac.TrackCondition.count()` — Returns number of conditions defined on the current track.
- `ac.TrackCondition.name(index)` — Returns name of a condition with certain index.
- `ac.TrackCondition.input(index)` — Returns input of a condition with certain index.
- `ac.TrackCondition.get(index, offset)` — Returns value of a condition with certain index.
- `ac.TrackCondition.getColor(index, offset)` — Returns RGB value of a condition with certain index.

### Ses (27)

- `ac.getSoundSpeedMs()` — Returns precalculated sound speed in m/s taking into account humidity, altitude, pressure, etc.
- `ac.getAudioVolume(audioChannelKey, carIndex, fallbackValue)` — Returns audio volume for given channel, value from 0 to 1. If channel is not recognized, returns `fallbackValue` if specified, unless (since 0.2.4) you specify key in a `
- `ac.getAudioOutputDevice()` — Returns name of output audio device.
- `ac.loadSoundbank(soundbank, guids)` — Loads FMOD soundbank. After soundbank is loaded, new audio events can be created which would refer to events in the soundbank. If soundbank is missing, new audio events c
- `ac.isAudioReady()` — Checks if audio engine is initialized and ready to work. Until then, all created audio events would return `false` in their `:isValid()` methods, it doesn’t mean that the
- `ac.setAudioEventMultiplier(surfaceWav, mult)` — Set volume multiplier for audio associated with a certain surface. Returns `false` if there are no events associated with this type of surface.
- `ac.AudioEvent.keepAlive()` — Deprecated, now all events are alive until `:dispose()` is called.
- `ac.setAudioVolume(audioChannelKey, value, carIndex)` — Sets audio volume for given channel, value from 0 to 1. If channel is not recognized, does nothing, unless (since 0.2.4) you specify key in a `'your.namespace/Readable na
- `ac.onAudioVolumeCalculation(callback)` — Sets a callback which will be called each frame when computing overall audio volume level. Callback returns a number which is used as volume multiplier. Use this function
- `ac.setExteriorAudioMix(value)` — Mix exterior audio for interior views. If your car has something like rolling down windows, might be a good idea to set it to 1 then.
- `ac.AudioEvent(event, reverbResponse, useOcclusion)` — Create a new audio event from previously loaded soundbank.
- `ac.AudioEvent.fromFile(params, reverbResponse)` — Create a new audio event from a file. Consequent calls with the same parameters would reuse previously loaded audio file. Note: if `loop` is set to `false` when used with
- `ac.AudioEvent.group(params, reverbResponse)` — Create a new audio event group for audio events created from files.
- `ac.CarAudioTweak.setVolume(eventID, value)` — Set volume multiplier. Overrides `[AUDIO_VOLUME]` value from car config.
- `ac.CarAudioTweak.getVolume(eventID)` — Get current volume multiplier set by `[AUDIO_VOLUME]` or a Lua script.
- `ac.CarAudioTweak.setPitch(eventID, value)` — Set pitch multiplier. Overrides `[AUDIO_PITCH]` value from car config.
- `ac.CarAudioTweak.getPitch(eventID)` — Get current pitch multiplier set by `[AUDIO_PITCH]` or a Lua script.
- `ac.CarAudioTweak.setDistanceMin(eventID, value)` — Set attenuation start distance (at this distance volume will start decreasing). Overrides `[AUDIO_3D_DISTANCE] …_MIN` value from car config.
- `ac.CarAudioTweak.getDistanceMin(eventID)` — Get current attenuation start distance set by audio event properties, `[AUDIO_3D_DISTANCE]` or a Lua script.
- `ac.CarAudioTweak.setDistanceMax(eventID, value)` — Set attenuation end distance (at this distance volume will stops decreasing, but sound might still be heard,  depending on FMOD audio event configuration). Overrides `[AU
- `ac.CarAudioTweak.getDistanceMax(eventID)` — Get current attenuation end distance set by audio event properties, `[AUDIO_3D_DISTANCE]` or a Lua script.
- `ac.CarAudioTweak.setTransform(eventID, value)` — Set audio transformation (position and rotation) relative to car. Overrides `[AUDIO_TRANSFORM]` value from car config.
- `ac.CarAudioTweak.setParameterTransform(eventID, key, value)` — Set LUT used for converting parameter values computed by AC into something else. Overrides `[AUDIO_PARAMETER_TRANSFORM]` value from car config.
- `ac.CarAudioTweak.setParameter(eventID, key, value, override)` — Set value of an FMOD audio event parameter. Feel free to call it every frame if you want to update it live.
- `ac.CarAudioTweak.setDSP(eventID, dsp, key, value)` — Set value of a custom DSP for the audio event. If DSP is missing, create and add it at the front.
- `ac.CarAudioTweak.getParameter(eventID, key)` — Get value of an FMOD audio event parameter.
- `ac.CarAudioTweak.replaceAudioEvent(eventID, newEventKey)` — Replaces entire audio event with a different one.

### Girdi / kontroller (21)

- `ac.getControllerSteerValue()` — Returns steering input from -1 to 1.
- `ac.isControllerGasPressed()` — Is gas input pressed (pedal, gamepad axis, keyboard button but not mouse button).
- `ac.isControllerGearUpPressed()` — Is gear up input pressed (pedal, gamepad button, keyboard button).
- `ac.isControllerGearDownPressed()` — Is gear down input pressed (pedal, gamepad button, keyboard button).
- `ac.isGamepadButtonPressed(gamepadIndex, gamepadButtonID)` — Checks if a certain gamepad button is pressed.
- `ac.getGamepadAxisValue(gamepadIndex, gamepadAxisID)` — Returns value of a certain gamepad axis.
- `ac.getJoystickButtonsCount(joystick)` — Returns number of buttons of a DirectInput device (ignore misleading name).
- `ac.isJoystickButtonPressed(joystick, button)` — Checks if a button of a DirectInput device is currently held down (ignore misleading name).
- `ac.onControlSettingsChanged(callback)` — Sets a callback which will be called when control settings change live.
- `ac.INIConfig.controlsConfig()` — Returns controls config (`cfg/controls.ini`). Returned file can’t be saved.
- `ac.getDualSenseControllers()` — Return table with gamepad indices for keys and 0-based indices of associated cars for values.
- `ac.getDualShockControllers()` — Return table with gamepad indices for keys and 0-based indices of associated cars for values.
- `ac.blockEscapeButton()` — Block Escape button from pausing AC for a few frames.
- `ac.onConsoleInput(callback)` — Sets a callback which will be called each time a new command is entered in AC console. If callback would return `true`, AC wouldn’t process command (return it if your scr
- `ac.reloadControlSettings()` — Reload control settings.
- `ac.overrideCarControls(carIndex)` — Access car controls overwriting. Easier than simulating inputs with fake gamepads and keypresses.
- `ac.loadCarControlsConfiguration(carIndex, relativeConfigName)` — Use carefully and in limited amount: currently each loaded configuration stays in memory, expects to override controls of an AI driver.
- `ac.setMousePosition(posPixels)` — Move mouse to a certain position.
- `ac.setMouseLeftButtonDown(isDown)` — Can be used for simulating mouse clicks.
- `ac.setMouseWheel(scroll)` — Can be used for simulating mouse wheel.
- `ac.ControlButton(id, defaults)`

### Araç özellikleri / ayarlar (89)

- `ac.getTyresName(carIndex, compoundIndex)` — Get short name of a tyre set, either currently selected or with certain index. If there is no such car, returns `nil`.
- `ac.getTyresIndex(carIndex, tyresShortName)` — Get 0-based index of a tyres set with a given short name, or -1 if there is no such tyres set (or such car).
- `ac.getTyresLongName(carIndex, compoundIndex, includePostfix)` — Returns long name of a tyre set with certain index. If there is no such car, returns `nil`.
- `ac.getCarID(carIndex)` — Get car ID (name of its folder) of a certain car. If there is no such car, returns `nil`.
- `ac.getCarName(carIndex, includeYearPostfix)` — Get car name (from its JSON file) of a certain car. If there is no such car, returns `nil`.
- `ac.getCarSkinID(carIndex)` — Get selected skin ID of (name of skin’s folder) of a certain car. If there is no such car, returns `nil`.
- `ac.getCarBrand(carIndex)` — Get name of a manufacturer of a certain car. If there is no such car, returns `nil`.
- `ac.getCarCountry(carIndex)` — Get name of manufactoring country of a certain car. If there is no such car, returns `nil`.
- `ac.getCarTags(carIndex)` — Get car tags. If there is no such car, returns `nil`. @nodiscard
- `ac.getCarDataFiles(carIndex)` — Returns ordered list of data file names (not full paths, just the names) of a certain car. Works for both packed and unpacked cars. If failed, returns empty list. @nodisc
- `ac.getCarColliders(carIndex, actualColliders)` — Returns list of car colliders. @nodiscard
- `ac.getCarSpeedKmh(carIndex)` — Consider using `ac.getCar(carIndex).speedKmh` instead.
- `ac.getCarGearLabel(carIndex)` — Returns 'R', 'N', number of engaged gear or value set by `ac.setGearLabel()` if used (for implementing automatic gearboxes). If your code is displaying current gear, this
- `ac.getDriverName(carIndex)` — Get full driver name of a driver of a certain car. If there is no such car, returns `nil`.
- `ac.getDriverNationCode(carIndex)` — Get three character nation code of a driver of a certain car. Nation code is a three-letter uppercase country identifier. If nationality is not set, a value from JSON is 
- `ac.getDriverNationality(carIndex)` — Get full nationality of a driver of a certain car. Usually, it’s a full country name. If nationality is not set, a value from JSON is returned. If it’s missing there, a f
- `ac.getDriverTeam(carIndex)` — Get name of a team of a driver of a certain car. Team names can be configured in entry list online. If nationality is not set, a value from JSON is returned. If it’s miss
- `ac.getDriverNumber(carIndex)` — Get number of a driver of a certain car. If number is set in skin JSON, it will be returned, otherwise it’s a unique 1-based number. If there is no car with such index, 0
- `ac.getCarByDriverName(driverName)` — Returns index of a car with a driver with a certain name, or -1 if there is no such car.
- `ac.getCarLeaderboardPosition(carIndex)` — Returns leaderboard car position, same as Python function with the same name. Does not work online. For an alternative solution, get position calculated by CSP via `ac.ge
- `ac.refreshCarShape(carIndex)` — Call this function if your script caused car shape to change and CSP would refresh interior masking, car heightmap and more.
- `ac.refreshCarColor(carIndex)` — Call this function if your script caused car color to change and CSP would refresh color map for bounced light and more.
- `ac.updateDriverModel(carIndex, forceVisible)` — Updates state of high-res driver model. Use it before moving driver nodes manually for extra animations: if called,  next model update possibly overwriting your custom po
- `ac.getDriverHeadTransformTo(ret, carIndex)`
- `ac.onTyresSetChange(carIndex, callback)` — Sets a callback which will be called when selected tyres change for some car. For car scripts to track changes in their own car, use `car.index` as first argument.  Note:
- `ac.getCarOptimalBrakingAmount(carIndex)` — Returns an estimate of optimal braking amount based on current grip level, or -1 if it can’t be computed for a given car.
- `ac.getCarMaxSpeedWithGear(carIndex, gearIndex)` — Returns an estimate of maximum speed for a given gear in km/h, -1 if it can’t be computed for a given car, or 0 if there is no such gear. Returns negative speed for the r
- `ac.getCarIndexInFront(carMainIndex, distance)` — Returns index of a car in front of other car (within 100 m), or -1 if there is no such car. Broken, not sure why is it even here, kept for compatibility.
- `ac.getGapBetweenCars(carMainIndex, carComparingToIndex)` — Calculates time gap between two cars in seconds. In race sessions uses total driven distance and main car speed, in other sessions simply compares best lap times. If main
- `ac.getCarSetupState()` — Returns player’s car setup state.
- `ac.areTyresLegal(name)` — Returns `false` if tyres with this short name are illegal in this race. Note: if all of car tyres are illegal, all of them will be legal.
- `ac.getCarBlindSpot(carIndex)` — Return distances to the nearest left and right cars in meters within 20 meters, or `nil` if there are no cars nearby or blind spot detection is not available for this car
- `ac.onCarJumped(carIndex, callback)` — Sets a callback which will be called when a car teleports somewhere or its state gets reset.
- `ac.onCarColorChanged(carIndex, callback)` — Sets a callback which will be called when a car’s color changes.
- `ac.onCarCollision(carIndex, callback)` — Sets a callback which will be called when a car collides with a wall or another car. Note: collisions in AC are usually lasting a few frames, but this callback will only 
- `ac.onSetupsListRefresh(callback)` — Sets a callback which will be called when list of setups changes.
- `ac.onSetupFile(callback)` — Sets a callback which will be called when setup is loaded or saved. Use `ac.INIConfig.currentSetup()` to load the setup.
- `ac.areCarsColliding(car0, car1)` — Works even if collisions are disabled, actually checks collider state live. Invokes ODE OPCODE computation, so don’t use too much.
- `ac.getCarState(index)` — Use `ac.getCar()` instead. Here, index starts with 1! With `ac.getCar()` index starts with 0, to match the rest of API functions
- `ac.getCarPhysics(index)` — Returns additional details on physics state of a car. Not available in replays or for remote cars. Updates once a frame at graphics rate. If you’re writing a physics scri
- `ac.INIConfig.carData(carIndex, fileName)` — Load car data INI file. Supports “data.acd” files as well. Returned files might be tweaked by things like custom physics virtual tyres. To get original file, use `ac.INIC
- `ac.INIConfig.carConfig(carIndex)` — Returns CSP config for a car. Might be slow: some of those configs are huge. Make sure to cache the resulting value if you need to reuse it. Returned file can’t be saved.
- `ac.INIConfig.currentSetup()` — Returns current setup INI file (either previously loaded or saved). Any changes in actual setup in pits won’t be reflected in the returned data unless file was saved. Ret
- `ac.DataLUT11.carData(carIndex, fileName)` — Load car data 1D-to-1D LUT file. Supports “data.acd” files as well.
- `ac.DataLUT21.carData(carIndex, fileName)` — Load car data 2D-to-1D LUT file. Supports “data.acd” files as well.
- `ac.DriverTags(driverName)` — Faster way to deal with driver tags. Any request of unsupported fields will return `false` for further extendability. Scripts with access to I/O can also alter fields.
- `ac.DriverTags(driverName)`
- `ac.getCar(index)` — Returns reference to a structure with various information about the state of a car. Very cheap to use. This is a new version with shorter name and 0-based indexing (to ma
- `ac.getCar.ordered(index)` — Returns Nth closest to camera car (pass 0 to get an ID of the nearest car). Inactive cars don’t count, so the number of cars here might be smaller than total number of ca
- `ac.getCar.leaderboard(index)` — Returns Nth car in the race leaderboard (uses lap times in practice and qualify sessions). Pass 0 to get the top one.
- `ac.getCar.serverSlot(index)` — Returns Nth car in server entry list. Pass 0 to get the first one. In offline races returns `nil`.
- `ac.iterateCars()` — Iterates over all the cars from one with 0th index to the last one. Use in a for-loop. To get a Nth car, use `ac.getCar()`. Example: for i, c in ac.iterateCars() do   ac.
- `ac.iterateCars.ordered(inverse)` — Iterates over active cars (excluding disconnected ones online) from nearest to furthest. Use in a for-loop. To get a Nth car, use `ac.getCar.ordered()`. Example: for i, c
- `ac.iterateCars.leaderboard()` — Iterates over cars from first to last in the race leaderboard (uses lap times in practice and qualify sessions). Use in a for-loop. To get a Nth car, use `ac.getCar.leade
- `ac.iterateCars.serverSlots()` — Iterates over cars based on their `sessionID` (index of a session slot). Use in a for-loop. To get a Nth car, use `ac.getCar.serverSlot()`. In offline races returns an em
- `ac.getCarConfig(carIndex, section, key, defaultValue)` — Reads a value from the config of a car. To use it, you need to specify `defaultValue` value, it would be used to determine the type of the value you need (and would be re
- `ac.setDriverChatNameColor(carIndex, color)` — Sets or resets custom driver color for CSP chat app.
- `ac.hideCarLabels(carIndex, hide)` — Toggles visibility of car labels.
- `ac.setDriverVisible(carIndex, isVisible)` — Toggles visibility of driver model. Meant to be used for debugging, but could also be used for screenshots, for example.
- `ac.setDriverMouthOpened(carIndex, amount)` — Sets how much driver mouth is opened. Requires properly configured human surface shader to have an effect.
- `ac.setDriverDoorOpen(carIndex, isOpen, instant)` — Toggles driver door animation.
- `ac.disableCarRecovery(disable)` — Disables car recovery (happens if car is flipped or falling through ground, use if for your mode or logic car might purposely end up in strange places and you don’t need 
- `ac.isCarResetAllowed()` — Returns `true` if resetting car is allowed in current session, invalidates lap.
- `ac.resetCar()` — Resets car back to the track. Available in few basic modes in offline only (such as practice with a single car), invalidates lap.
- `ac.saveCarStateAsync(callback)` — Save car state. Saves a lot of stuff, including velocities, momentum, engine and gearbox state, damage levels. Because of that save from one car can’t be applied to a dif
- `ac.loadCarState(data, dataNext, blend, iterations)` — Load car state. Available in few basic modes in offline only (such as practice with a single car), invalidates lap.
- `ac.setCarActive(carIndex, active)` — Toggles car visibility.
- `ac.configureCarForceFeedback(carIndex, ffbBase, ffbSteerAssist)` — Alter `FFMULT` and `STEER_ASSIST` from car.ini. Original values are available in car state.
- `ac.replaceDriverModel(carIndex, data)` — Replace driver model with a custom one asynchronously. For method to work current and new driver models should have common skeleton.
- `ac.packDriverModel(destination, lodAData, lodBData, configData)` — Packs driver model into a replacement package.
- `ac.overrideTyreSmoke(tyreIndex, intensity, thickness, surfaceHeat)` — Changes behaviour of smoke going off a tyre (won’t affect dust or water splashes). Actual movement of smoke might change with CSP updates.
- `ac.setGearLabel(c)` — Changes label used to display current gear. Label can be up to four symbols long. If label is 2 characters or less, it’ll be stored in a replay as well (do any gearboxes 
- `ac.setBrakeBias(balance)`
- `ac.setEngineBrakeSetting(settingIndex)`
- `ac.switchToNeutralGear()`
- `ac.setTargetCar(carIndex)` — Changes index of a target car. Target car is used by car controlling API functions which don’t receive car index, such as `ac.setDRS()`. Originally those functions were i
- `ac.redirectDamageDisplayer(active)`
- `ac.redirectFuelWarningIndicator(active)`
- `ac.getSetupSpinners()` — Get information about available spinners in setup menu. Names match section names of setup INI files. Value `label` might contain localized setup items. Array `itemValues
- `ac.isSetupAvailableToEdit()`
- `ac.getSetupSpinnerValue(setupValueID, fallbackValue)`
- `ac.setSetupSpinnerValue(setupValueID, value)` — Returns `false` if there is no such element or it can’t be edited now.
- `ac.refreshSetups()` — Refreshes list of setups. Any listeners set with `ac.onSetupsListRefresh()` will be called as well.
- `ac.saveCurrentSetup(setupFilename)` — Saves current setup into a file. If successful, refreshes setups list and calls any listeners set with `ac.onSetupsListRefresh()` as well.
- `ac.setActiveSetupName(setupName, trackID, changed)` — Change currently active setup name used for saving setups.
- `ac.loadSetup(setupFilename)` — Loads car setup from a file. Works only from setup menu, and only if setup is not fixed.
- `ac.resetSetupToDefault()` — Resets car setup to default values. Works only from setup menu, and only if setup is not fixed.
- `ac.stringifyCurrentSetup()` — Exports current setup into a string.
- `ac.DriverWeightShift(carIndex)` — Access driver weight shift.

### Oturum / yarış / ağ (42)

- `ac.isInReplayMode()`
- `ac.getSessionName(sessionIndex)` — Get session name for a session with given index. Use `ac.getSim()` to check number of sessions and more information about them. If there is no such session, returns `nil`
- `ac.getSessionSpawnSet(sessionIndex)` — Get session spawn set (`'START'`, `'PIT'`, `'HOTLAP_START'`, `'TIME_ATTACK'`, etc.) for a session with given index. Use `ac.getSim()` to check number of sessions and more
- `ac.getServerName()` — Returns name of the current online server, or `nil` if it’s not available.
- `ac.getServerIP()` — Returns IP address of the current online server, or `nil` if it’s not available.
- `ac.getServerPortHTTP()` — Returns HTTP post of the current online server, or -1 if it’s not available.
- `ac.getServerPortTCP()` — Returns TCP post of the current online server, or -1 if it’s not available.
- `ac.getServerPortUDP()` — Returns UDP post of the current online server, or -1 if it’s not available.
- `ac.blockSystemMessages(regex)` — Block system messages based on a given regular expression.
- `ac.onSessionStart(callback)` — Sets a callback which will be called when a new session starts (or restarts). Doesn’t get triggered at the start of the race (Lua scripts load when the first session is a
- `ac.setMessage(title, description, type, time)` — Show message using AC system messages UI. Pass empty `title` and `description` to hide currently shown message, if any.  Types:  - Default: regular message shown on top i
- `ac.onMessage(callback)` — Listen to messages, including the ones shown by `ac.setMessage()` and `ac.setIllegalMessage()`.
- `ac.setSystemMessage(title, description)` — Use `ac.setMessage()` instead.
- `ac.getUnreadChatMessages()` — Return the number of unread chat messages, or 0 in an offline race.
- `ac.onLapCompleted(carIndex, callback)` — Sets a callback which will be called when a car completes a lap. Should work for remote cars as well.
- `ac.getSession(index)` — Returns reference to a structure with various information about certain session. Very cheap to use. Note: not all data  might be available online.
- `ac.onOnlineWelcome(callback)` — Sets a callback which will be called when server welcome message and extended config arrive.
- `ac.INIConfig.onlineExtras()` — Returns config with extra online options, the ones that can be set with Content Manager. Returned file can’t be saved.
- `ac.INIConfig.raceConfig()` — Returns race config (`cfg/race.ini`). Password and online GUID won’t be included. Returned file can’t be saved.
- `ac.OnlineEvent(layout, callback, namespace, udp, params)` — Creates a new type of online event to exchange between scripts running on different clients in an online race. Example: local chatMessageEvent = ac.OnlineEvent({   -- mes
- `ac.writeReplayBlob(key, data)` — Writes additional data to replay. Use `ac.readReplayBlob()` to extract data later in replay mode. Data written this way is not tied to frames. Don’t bother compressing da
- `ac.readReplayBlob(key)` — Reads additional data from replay if recorded with `ac.writeReplayBlob()`.
- `ac.ReplayStream(layout, callback, frameDivisor)` — Create a new stream for recording data to replays. Write data in returned structure if not in replay mode, read data if in replay mode (use `sim.isReplayActive` to check 
- `ac.checkIfServerRequiresPasswordAsync(callback)` — Checks if server requires a password to join. Asyncronous version. Returns error (via `err` callback parameter) if this is not an online race.
- `ac.disableQuickMenuPitstop(disable)` — Disables quick pitstop menu.
- `ac.setRaceScore(carIndex, score)` — Override race score used for leaderboard sorting.
- `ac.sendChatMessage(command)` — Sends a new chat message. Not available to scripts without I/O access.
- `ac.onChatMessage(callback)` — Sets a callback which will be called each time a new chat message arrives. Return `true` from callback to handle chat message and stop it from showing in chat apps. Secon
- `ac.markLapAsSpoiled(silent)` — Marks current lap done by player as spoiled. Works only during the race
- `ac.tryToRestartSession()` — Attempts to restart current session (for offline racing only).
- `ac.tryToSkipSession()` — Attempts to restart current session (for offline racing only).
- `ac.tryToOpenRaceMenu(tab)` — Attempts to open race menu (only available if car is in pits).
- `ac.tryToTeleportToPits()` — Teleports car to pits.
- `ac.overrideReplayConditions(set)` — Override weather conditions in replay. Can be called each frame for smooth transition.
- `ac.tryToToggleReplay(active, rewindS)` — Starts or stops instant replay.
- `ac.setReplayPosition(frameIndex, playCounter)` — Sets index of a current replay frame.
- `ac.accessOverlayLeaderboardParams()`
- `ac.teleportToServerPoint(pointIndex)` — Teleports car to a point defined by a server. Available online and only with servers with custom teleport destinations. Usual restrictions still apply. Meant for creating
- `ac.canTeleportToServerPoint(pointIndex)` — Checks if it’s possible to teleport to a custom server teleport destination. Meant for creating custom UI for that function. Use `ac.INIConfig.onlineExtras()` to get serv
- `ac.getPitstopSpinners()` — Added in CSP 0.2.7, collects information about spinners in Quick Pitstop menu. Entries are listed in order they’re shown in UI. Each entry has `value` storing value set i
- `ac.setPitstopSpinnerValue(valueName, value, presetIndex)` — Returns `false` if there is no such element or it can’t be edited now.
- `ac.setCurrentQuickPitPreset(preset)` — Change currently selected Quick Pit preset. Use `ac.getSim().currentQuickPitPreset` to get the currently selected presets index (0-based).

### Depolama / dosya / yapılandırma (36)

- `ac.loadINIppFile(iniFilename, includeDirs)` — Load and parse INIpp configuration file (supports includes and such), return it as JSON. Deprecated, use `ac.INIConfig.load()` instead.
- `ac.findFile(fileName)` — If `fileName` is not an absolute path, looks for a file in script directory, then relative to CSP folder, then relative to AC root folder. If anything is found, returns a
- `ac.getFolder(folderID)` — Returns full path to one of known folders. Some folders might not exist, make sure to create them before writing.
- `ac.readDataFile(filename)` — Can be used to access files in “data.acd” and, for example, read car specs.
- `ac.parseINIppFile(iniData)` — Parse INIpp configuration file, return it as JSON. Deprecated, use `ac.INIConfig.parse()` instead.
- `ac.onClientConnected(callback)` — Sets a callback which will be called when new user connects the server and their car appears (doesn’t do anything outside of online race).
- `ac.onClientDisconnected(callback)` — Sets a callback which will be called when a user disconnects (doesn’t do anything outside of online race).
- `ac.onTripleConfigurationChanged(callback)` — Sets a callback which will be called when triple screen configuration changes.
- `ac.storageSetPath(relativePath, prefix)` — Changes location of script storage. Scripts without I/O access will have “shared-” appended to their relative path and can’t use “.” or slashes in it to ensure configs ar
- `ac.store(key, value)` — Stores value in session shared Lua/Python storage. This is not a long-term storage, more of a way for different scripts to exchange data. Note: if you need to exchange a 
- `ac.load(key)` — Reads value from session shared Lua/Python storage. This is not a long-term storage, more of a way for different scripts to exchange data. Note: if you need to exchange d
- `ac.onFileChanged(filename, callback)` — Sets a callback which will be called when a file changes. Not available to scripts without I/O access.
- `ac.onFolderChanged(folder, filter, recursive, callback)` — Sets a callback which will be called when folder contents change. Not available to scripts without I/O access.
- `ac.onCSPConfigChanged(cspModuleID, callback)` — Sets a callback which will be called when config for certain CSP module has changed.
- `ac.getTripleConfiguration()` — Returns data regarding current triple screen configuration. If AC is not launched in triple screen mode, returns `nil` instead.
- `ac.INIConfig(format, sections)` — A wrapper for data parsed from an INI files, supports different INI formats. Parsing is done on CSP side, rest is on CSP side. Use `:get()` and `:set()` methods to operat
- `ac.INIConfig.parse(data, format, includeFolders)` — Parse INI config from a string.
- `ac.INIConfig.load(filename, format, includeFolders)` — Load INI file, optionally with includes.
- `ac.INIConfig.videoConfig()` — Returns video config (`cfg/video.ini`). Returned file can’t be saved.
- `ac.INIConfig.cspModule(cspModuleID)` — Load config of a CSP module by its name.
- `ac.INIConfig.scriptSettings()` — Load config of the current Lua script (“settings.ini” in script directory and settings overriden by user, meant to be customizable with Content Manager).
- `ac.DataLUT11.load(filename)` — Load 1D-to-1D LUT file.
- `ac.DataLUT21.load(filename)` — Load 2D-to-1D LUT file.
- `ac.connect(layout, keepLive, namespace)` — Creates a new shared structure to quickly exchange data between different Lua scripts within a session. Example: local sharedData = ac.connect{   ac.StructItem.key('myCha
- `ac.readMemoryMappedFile(filename, layout, persist)` — Opens shared memory file for reading. Do not attempt to modify any of its contents: doing so pretty much always would result in Assetto Corsa just straight up crashing.
- `ac.writeMemoryMappedFile(filename, layout, persist)` — Opens shared memory file for writing. Note: if the file would exist at the moment of opening (for example, created before by a different Lua script, or by a separate proc
- `ac.disposeMemoryMappedFile(reference)` — Forcefully closes memory mapped file opened either for reading or writing without waiting for GC to pick it up.
- `ac.storage(layout, keyPrefix)` — Storage function. Easiest way to use is to pass it a table with default values — it would give you a table back which would load values on reads and save values on writes
- `ac.storageHasKey(storage, key)` — Checks if storage table created by `ac.storage(table)` has a certain key or not.
- `ac.storageChanged()` — Returns `true` if any storage value has changed since the previous call.
- `ac.applyContentConfig(targetIndex, data, flags)` — Similar to `ac.ext_applyTrackConfig()` from CSP Python API, reload a track config live. Works even if live updates are disabled. Use carefully, might cause a freeze or cr
- `ac.pauseFilesWatching(pause)` — Pauses or un-pauses file system monitoring used for live reloading. Might be useful to pause it if you’re about to update a lot of configs, for example. Make sure to alwa
- `ac.reloadPythonApp(appName, withoutDisposing)` — Tries to reload a Python app with given name (name of its folder).
- `ac.applyLiveConfigEdit(config, data)` — Applies an edit to AC config, such as “video.ini”, live without actually altering file on a disk. Only a very few values are currently supported.
- `ac.setVRSConfigurationForVR(active, rateOuter, rateMiddle, rateInner, areaOuter, areaMiddle, areaInner)`
- `ac.unloadApp()` — Fully unloads and terminates current app if possible. Execution will continue after this function, unloading will be attempted in the next frame. Only apps with no visibl

### Diğer (216)

- `ac.getPatchVersion()`
- `ac.getPatchVersionCode()` — Increments with every CSP build.
- `ac.getMGUKDeliveryName(carIndex, programIndex)` — Returns name of MGUK delivery program. If there is no such car or program, returns `nil`.
- `ac.dirname()` — Returns directory of the script.
- `ac.getCountryName(nationCode)` — Returns country name based on nation code (three symbols for country ID).
- `ac.uniqueMachineKey()` — A key unique for each individual PC (uses serial numbers of processor and motherboard). Use `ac.uniqueMachineKeyAsync()` instead.
- `ac.uniqueMachineKeyAsync(callback)` — A key unique for each individual PC (uses serial numbers of processor and motherboard). Asynchronous version. Returned value is in binary, use something like `ac.encodeBa
- `ac.distanceToRenderSquared(pos)` — Returns squared distance to main camera. In splitscreen mode, returns smallest distance between to cameras. In the future, might be altered by more entities, such as addi
- `ac.distanceToRender(pos)` — Returns distance to main camera. In splitscreen mode, returns smallest distance between to cameras. In the future, might be altered by more entities, such as additional O
- `ac.closerToRenderThan(pos, distance)` — Returns `true` if position is closer to main camera than the threshold. In splitscreen mode, considers secondary camera. In the future, might be altered by more entities,
- `ac.encodeHalf(v)` — Encodes float into FP16 format and returns it as uint16.
- `ac.encodeHalf2(v)` — Encodes two floats from a vector into FP16 format and returns it as uint32.
- `ac.decodeHalf(v)` — Decodes float from FP16 format (represented as uint16) and returns a regular number.
- `ac.decodeHalf2(v)` — Decodes two floats from FP16 format (represented as uint32) and returns a vector.
- `ac.checksumSHA256(data)` — Computes SHA-256 checksum for given binary data. Very secure, but might be slow with large amounts of data. Data string can contain zero bytes.
- `ac.checksumXXH(data)` — Computes 64-bit xxHash checksum for given binary data. Very fast, not that great for encryption purposes. Use `bit.tohex()` to turn result into a hex representation. Data
- `ac.compress(data, type, level)` — Compresses data. First byte of resulting data is compression type, next four are uncompressed data size, rest is compressed data itself. If data is failed to compress, re
- `ac.decompress(data)` — Decompresses data. First byte of input data is compression type, next four are uncompressed data size. If data is damaged, returns `nil`. Data string can contain zero byt
- `ac.encodeBase64(data, trimResult)` — Encodes data in base64 format. Data string can contain zero bytes.
- `ac.decodeBase64(data)` — Decodes data from base64 format (ending “=” are not needed).
- `ac.utf8To16(data)` — Converts string from UTF-8 to UTF-16 format (two symbols per character). All strings Lua operates with regularly are consired UTF-8. UTF-16 strings can’t be used in any C
- `ac.utf16To8(data)` — Converts string from UTF-16 (two symbols per character) to common Lua UTF-8. All strings Lua operates with regularly are consired UTF-8. UTF-16 strings can’t be used in a
- `ac.structBytes(data)` — Given an FFI struct, returns bytes with its content. Resulting string may contain zeroes.
- `ac.memoryBarrier()`
- `ac.perfBegin(value)` — Simple helper to measure time and analyze performance. Call `ac.perfBegin('someKey')` to start counting time and  `ac.perfEnd('someKey')` to stop. Measured time will be s
- `ac.perfEnd(value)` — Simple helper to measure time and analyze performance. Call `ac.perfBegin('someKey')` to start counting time and  `ac.perfEnd('someKey')` to stop. Measured time will be s
- `ac.perfFrameBegin(value)` — Unlike `ac.perfBegin('someKey')/ac.perfEnd('someKey')`, `ac.perfFrameBegin(0)/ac.perfFrameEnd(0)` will accumulate time  between calls as frame progresses and then use the
- `ac.perfFrameEnd(value)` — Unlike `ac.perfBegin('someKey')/ac.perfEnd('someKey')`, `ac.perfFrameBegin(0)/ac.perfFrameEnd(0)` will accumulate time  between calls as frame progresses and then use the
- `ac.getCompassAngle(dir)` — Returns compass angle for given directory.
- `ac.isInteriorView()` — Returns true if camera is focused on interior (interior audio is playing).
- `ac.compressTexture(filename, outputFilename)`
- `ac.getAirPressure(p)` — Returns air pressure in kPa.
- `ac.getAirHumidity(p)` — Returns air humidity in 0…1 range. Currently doesn’t use position parameter, but it might change later.
- `ac.getLastError()` — Returns string with last error thrown by this script, or `nil` if there wasn’t an error. Use it in case you would want to set some nicer error reporting.
- `ac.getGroundYApproximation()` — Returns approximate Y coordinate of ground, calculated by using depth from reflection cubemap. Does not have a performance impact (that value  will be calculated anyway f
- `ac.getDeltaT()` — Returns current delta time associated with UI (so values are non-zero if sim or replay are paused).
- `ac.getGameDeltaT()` — Returns current delta time associated with simulation (so values are zero if sim or replay are paused).
- `ac.getScriptDeltaT()` — Returns delta time for current script. If script only runs every N frames (like car display scripts by default), this value will be greater than regular `dt` from simulat
- `ac.getPpFilter()` — Returns name of current PP filter with “.ini”.
- `ac.isCustomIconSet(carIndex)` — Returns `true` if a certain driver is using custom icon. Use `'carN::special::driver'` as image filename to draw it (when not custom, will give you a livery icon or a fal
- `ac.isKeyDown(keyIndex)` — Is keyboard button being held.
- `ac.isKeyPressed(keyIndex)` — Is keyboard button just moved from not held to held in the last graphics frame (or, for physics scripts since 0.2.7, physics frame). Still, if you’re working on car physi
- `ac.isKeyReleased(keyIndex)` — Is keyboard button just moved from held to not held in the last graphics frame (or, for physics scripts since 0.2.7, physics frame). If you’re working on car physics scri
- `ac.getJoystickCount()` — Returns number of DirectInput devices (ignore misleading name).
- `ac.getJoystickName(joystick)` — Returns name of a DirectInput device (ignore misleading name). If there is no such device, returns `nil`.
- `ac.getJoystickInstanceGUID(joystick)` — Returns instance GUID of a DirectInput device (ignore misleading name). If there is no such device, returns `nil`.
- `ac.getJoystickProductGUID(joystick)` — Returns product GUID of a DirectInput device (ignore misleading name). If there is no such device, returns `nil`.
- `ac.getJoystickIndexByInstanceGUID(guid)` — Returns index of a DirectInput device by its instance GUID, or `nil` if there is no such device (ignore misleading name).
- `ac.getJoystickAxisCount(joystick)` — While this function returns accurate number of device axis, consider using 8 instead if you need to iterate over them. Actual axis can be somewhere within those 8. For ex
- `ac.getJoystickDpadsCount(joystick)` — Returns number of D-pads (aka POVs) of a DirectInput device (ignore misleading name).
- `ac.getJoystickAxisValue(joystick, axis)` — Returns axis value of a DirectInput device (ignore misleading name).
- `ac.isJoystickAxisValue(joystick, axis)` — Use `ac.getJoystickAxisValue()` instead.
- `ac.getJoystickDpadValue(joystick, dpad)` — Returns D-pad (aka POV) value of a DirectInput device (ignore misleading name).
- `ac.isJoystickDpadValue(joystick, dpad)` — Use `ac.getJoystickDpadValue()` instead.
- `ac.getPenPressure()` — Checks current stylus/pen/mouse using RealTimeStylus API (compatible with Windows Ink). Should support things like Wacom tables (if drivers are installed and Windows Ink 
- `ac.getMoonFraction()` — How much of moon area is currently lit up.
- `ac.getAltitude()`
- `ac.onResolutionChange(callback)` — Sets a callback which will be called when AC resolution changes. Happens when window size changes, or when making a screenshot. Note: some scripts, such as WeatherFX styl
- `ac.isLuaAppRunning(appID)` — Returns `true` if a Lua app with given ID is loaded and currently active (not countring background services).
- `ac.broadcastSharedEvent(key, data)` — Broadcasts a shared event. With shared events, different Lua scripts can exchange messages and data. Make sure to come up with a unique name for your events to avoid coll
- `ac.onSharedEvent(key, callback, processPostponed)` — Subscribes to a shared event. With shared events, different Lua scripts can exchange messages and data. Make sure to come up with a unique name for your events to avoid c
- `ac.onLuaScriptDisposal(callback)`
- `ac.setVRHandBusy(hand, busy)`
- `ac.setVRHandVibration(hand, frequency, amplitude, duration)`
- `ac.isModuleActive(cspModuleID)`
- `ac.mediaCurrentPeak(output)` — Returns audio peak level for the system, for left and right channels. Careful: AC audio is also included, but it still might be used to fake some audio visualization.
- `ac.setTextureKey(key)` — Set texture key to load encoded textures.
- `ac.encodeTexture(filename, outputFilename, key, applyLz4Compression)` — Encode texture. To load texture later, first call `ac.setTextureKey()`.
- `ac.onAlbumCoverUpdate(callback)` — Sets a callback which will be called when album cover changes.
- `ac.mediaPlayPause()` — Pauses or unpauses current track in currently active music player (by simulating media key press).
- `ac.console(message, withoutPrefix)` — Prints message to AC console.
- `ac.setClipboardText(text)` — Sets given text to the clipboard.
- `ac.onOpenMainMenu(callback)` — Sets a callback which will be called when a command opening a certain section in main menu has been called. More types of pages might be added later.
- `ac.areShapesColliding(shape1, transform1, shape2, transform2)` — Check collisions between two shapes. Note: if you want to do something like check a car against a million of powerups on a track, consider doing some prefiltering to spee
- `ac.setLogSilent(value)` — Stops functions like `ac.log()` from logging things into CSP log file, in case you need to log a lot. With it, you  can use Lua Debug app to see latest log entries.
- `ac.clearDebug(filter)` — Removes `ac.debug()` entries matching filter.
- `ac.getConditionsSet()` — Returns original weather conditions without any filtering or sanity checks. For a faster and more filtered data check `ac.getSim().weatherConditions`.
- `ac.getConditionsSetTo(r)` — Returns original weather conditions without any filtering or sanity checks. For a faster and more filtered data check `ac.getSim().weatherConditions`.
- `ac.getSimState()` — Use `ac.getSim()` instead
- `ac.getUiState()` — Use `ac.getUI()` instead
- `ac.getSim()` — Returns reference to a structure with various information about the state of Assetto Corsa. Very cheap to use.  This is a new version with shorter name. Updates once per 
- `ac.getUI()` — Returns reference to a structure with various information about the state of the UI. Very cheap to use. This is a new version with shorter name. Updates once per graphics
- `ac.getVR()` — Returns reference to a structure with VR-related values, like hands and head positions. Very cheap to use. Note: currently, accurate values are available with Oculus only
- `ac.getDualSense(gamepadIndex)` — Returns extras of PS5 DualSense gamepad, such as accelerometer, gyroscope or battery state. Accelerometer and gyroscope values might be different from values reported by 
- `ac.setDualSense(gamepadIndex, priority, holdFor)` — Returns a structure with state of PS5 DualSense LEDs, change it to alter its state. Changes remain for some time, keep calling it for continuos adjustments. Note: if you’
- `ac.getDualShock(gamepadIndex)` — Returns extras of PS4 DualShock gamepad (or Nintendo gamepads), such as accelerometer, gyroscope or battery state. Accelerometer and gyroscope values might be different f
- `ac.setDualShock(gamepadIndex, priority, holdFor)` — Returns a structure with state of PS4 DualShock (or Nintendo) LEDs, change it to alter its state. Changes remain for some time, keep calling it for continuos adjustments.
- `ac.setXbox(gamepadIndex, priority, holdFor)` — Returns a structure with state of Xbox One gamepad, change it to alter its state. Changes remain for some time, keep calling it for continuos adjustments. Note: if you’re
- `ac.debug(key, value)` — Displays value in Lua Debug app, great for tracking state of your values live.
- `ac.log(...)` — Prints a message to a CSP log and to Lua App Debug log. To speed things up and only use Lua Debug app, call `ac.setLogSilent()`.
- `ac.warn(...)` — Prints a warning message to a CSP log and to Lua App Debug log. To speed things up and only use Lua Debug app, call `ac.setLogSilent()`.
- `ac.error(...)` — Prints an error message to a CSP log and to Lua App Debug log. To speed things up and only use Lua Debug app, call `ac.setLogSilent()`.
- `ac.fillStructWithBytes(destination, data)` — Given an FFI struct and a string of data, fills struct with that data. Works only if size of struct matches size of data. Data string can contain zeroes.
- `ac.stringToFFIStruct(src, dst, size)` — Fills a string of an FFI struct with data up to a certain size. Make sure to not overfill the data.
- `ac.onRelease(callback, item)` — Adds a callback which might be called when script is unloading. Use it for some state reversion, but don’t rely on it too much. For example, if Assetto Corsa would crash 
- `ac.DataLUT11()` — Creates a new empty 1D-to-1D LUT. Use `ac.DataLUT11:add(input, output)` to fill it with data.
- `ac.DataLUT11.parse(data)` — Parse 1D-to-1D LUT from a string in “(|Input1=Output1|Input2=Output2|…|)” format.
- `ac.DataLUT21()` — Creates a new empty 2D-to-1D LUT. Use `ac.DataLUT21:add(input, output)` to fill it with data.
- `ac.DataLUT21.parse(data)` — Parse 2D-to-1D LUT from a string in “(|X1,Y1=Output1|X2,Y2=Output2|…|)” format.
- `ac.StructItem.combine(layout, compact)` — Create a new struct from a given layout. Could be used in calls like `ac.structBytes()` and `ac.fillStructWithBytes()`. Each call defines and creates a new struct, so don
- `ac.StructItem.key(key)` — Adds a key to a structure to ensure its uniqueness. Consider using something like “yourUsername.yourContentID” or something like that for a key, so that your data would n
- `ac.StructItem.explicit(alignment, packing)` — Enables explicit ordering for your structures. By default, CSP will reorder fields in your structures for optimal data packing. Because all scripts written in Lua share t
- `ac.StructItem.half()`
- `ac.StructItem.float()`
- `ac.StructItem.double()`
- `ac.StructItem.norm8()`
- `ac.StructItem.unorm8()`
- `ac.StructItem.norm16()`
- `ac.StructItem.unorm16()`
- `ac.StructItem.int16()`
- `ac.StructItem.uint16()`
- `ac.StructItem.int32()`
- `ac.StructItem.uint32()`
- `ac.StructItem.int64()`
- `ac.StructItem.uint64()`
- `ac.StructItem.boolean()`
- `ac.StructItem.char()` — Same as `ac.StructItem.int8()`.
- `ac.StructItem.byte()` — Same as `ac.StructItem.uint8()`.
- `ac.StructItem.int8()`
- `ac.StructItem.uint8()`
- `ac.StructItem.vec2()`
- `ac.StructItem.vec3()`
- `ac.StructItem.vec4()`
- `ac.StructItem.rgb()`
- `ac.StructItem.rgbm()`
- `ac.StructItem.hsv()`
- `ac.StructItem.quat()`
- `ac.StructItem.array(elementType, size)`
- `ac.StructItem.struct(fields)`
- `ac.StructItem.string(capacity)`
- `ac.StructItem.mat3x3()`
- `ac.StructItem.mat4x4()`
- `ac.StructItem.transform(compactPosition, compactRotation, rangeFrom, rangeTo)` — Matrix packed to 6, 9 or 12 bytes (depending on settings). Note: to update value you need to use assignment operator (`.field = newValue`), altering matrix of this proper
- `ac.StructItem.build(layout)` — Simply create a new FFI structure with a given layout. Doesn’t connect to anything.
- `ac.HashSpace(cellSize)`
- `ac.Lut(data, hsvColumns)`
- `ac.isTaggedAsFriend(driverName)` — Checks if a user is tagged as a friend. Uses CSP and CM databases. Deprecated, use `ac.DriverTags` instead.
- `ac.tagAsFriend(driverName, isFriend)` — Tags user as a friend (or removes the tag if `false` is passed). Deprecated, use `ac.DriverTags` instead.
- `ac.currentlyPlaying()` — Syncs information about currently playing music and returns a table with details. Takes data from Windows 10 Media API, or from other sources configured in Music module o
- `ac.ConditionsSet()`
- `ac.HashSpace(cellSize)` — Simple structure meant to speed up collision detection by arranging items in a grid using hashmap. Cells are arranged horizontally.
- `ac.isTaggedAsFriend(driverName)` — Checks if a user is tagged as a friend. Uses CSP and CM databases. Deprecated, use `ac.DriverTags` instead.
- `ac.tagAsFriend(driverName, isFriend)` — Tags user as a friend (or removes the tag if `false` is passed). Deprecated, use `ac.DriverTags` instead.
- `ac.VertexBuffer(size)` — Buffer with mesh vertices. Contains `ac.MeshVertex` items.
- `ac.IndicesBuffer(size)` — Buffer with mesh indieces. Contains `integer` items (limited by 16 bits for AC to handle).
- `ac.findAny(s)` — Creates a new scene reference containing objects (nodes, meshes, etc.) collected with a filter from root node associated with current script. For most scripts it would be
- `ac.findByClass(objectClass, s)` — Creates a new scene reference containing objects of a certain class collected with a filter from root node associated with current script. For most scripts it would be an
- `ac.GeometryShot(sceneReference, resolution, mips, withDepth, antialiasingMode, textureFormat, flags)`
- `ac.getFFBTweaksReport()` — FFB (phase 1), FFB (phase 2), damper.
- `ac.setAppOpen(appID)` — Opens a main window of a Lua app (or, if opened, brings it to focus).
- `ac.checkAdminPrivileges()` — Refreshes `ac.SimState.isAdmin` flag. Returns `false` if another check is already in progress.
- `ac.recenterVR()`
- `ac.restartAssettoCorsa(raceIni)` — Restart Assetto Corsa (shut it down, but before that, add a flag to “race.ini” telling launcher to restart simulation).
- `ac.shutdownAssettoCorsa()` — Shutdown Assetto Corsa. Not available to scripts without I/O access.
- `ac.setBodyDirt(carIndex, dirt)`
- `ac.setReverbValue(reverbValue)`
- `ac.takeAStepBack(offset)` — Resets car back to the track and moves it a bit back. Available in few basic modes in offline only (such as practice with a single car), invalidates lap.
- `ac.tryToStart(instant)` — Tries to start the race (pretty much, just to press Start button in pits menu).
- `ac.tryToPause(active)` — Pauses or unpauses simulation (if possible; for example, it wouldn’t be paused online with moving car or when displaying results).
- `ac.trySimKeyPressCommand(command, shift)` — Runs a sim command usually associated with AC hotkeys, such as “Ideal Line”. Original hotkey logic is applied: for example, most commands won’t work if the race is not st
- `ac.setPhysicsDebugLines(mask, carsMask)` — Enables and disables physics debugging outlines. Pass `1` as `carsMask` to show lines for the first car only, `0` to show lines for all cars or set specific bits to 1 for
- `ac.setExposureMultiplier(value)`
- `ac.disableFrameLimiter(disable)` — Pause frame limiter (both vertical syncing and regular frame limiter). Temporary change, rolls back when script is unloaded.
- `ac.isPythonAppActive(appName)` — Checks if a Python app with given name (name of its folder) is currently loaded in-game.
- `ac.setVAOMode(mode)` — Sets current VAO mode, meant to be used for debugging purposes only.
- `ac.getVAOMode()` — Gets current VAO mode, meant to be used for debugging purposes only.  Use `ac.getSim().currentVAOMode` instead.
- `ac.setWiperSpeed(carIndex, wiperSpeed)` — Use `ac.setWiperMode()` instead.
- `ac.setHShifterActive(active)` — Toggle H-shifter to be active or not. Doesn’t have an effect if AC is not controlled by a wheel. Use `ac.getSim().controlsWithShifter` to check the current state.
- `ac.refreshVideoSettings()`
- `ac.consoleExecute(command)` — Executes command in AC console.
- `ac.simulateCustomHotkeyPress(key, frames)` — Simulate CSP hotkey (one of ones listed in Patch section of control settings in CM) press for the next few frames. Keep calling regularly if you need to hold down a butto
- `ac.resetVRPose()`
- `ac.setFFBMultiplier(multiplier)` — Changes FFB gain. Unlike `ac.setCarFFB()` in Python apps, this one takes an absolute value, not a relative offset. To get current FFB multiplier, use `ac.getCar(0).ffbMul
- `ac.setCurrentDesktop(desktop)`
- `ac.setVRSGazeForVR(fov, eyeL, eyeR)` — Returns `false` if gaze is not available.
- `ac.setHeatShimmerIntensity(key, intensity)` — Sets intensity of a heat shimmer zone with a given key. Returns `false` if there is no zone with such key.
- `ac.setScriptFlamesIntensity(intensity)` — Sets intensity of script flames. See docs for exhaust flames configuration on how to set them up.
- `ac.getExtraSwitchName(index)` — Returns name of a switch, or `nil` if switch name is not available or not named.
- `ac.setTurboWastegate(wastegate, turboIndex)` — Set a custom value for turbo wastegate. Accessible only if turbo is adjustable.
- `ac.getRealMirrorCount()` — Get number of Real Mirror mirrors in a car (for car scripts, associated car, for apps and tools — player’s car). If Real Mirrors are disabled or not active for that car, 
- `ac.getRealMirrorAABB(mirrorIndex, min, max)`
- `ac.setRealMirrorParams(mirrorIndex, params, userDriven)` — Updates mirror settings for a given Real Mirror mirror (for car scripts, associated car, for apps and tools — player’s car). Easiest way is to use it in combination with 
- `ac.setABS(mode)`
- `ac.setTC(mode)`
- `ac.isTurboWastegateAdjustable(turboIndex)`
- `ac.setDRS(state)`
- `ac.setKERS(state)`
- `ac.setMGUHCharging(charging)`
- `ac.setMGUKDelivery(level)`
- `ac.setMGUKRecovery(level)`
- `ac.setWiperMode(wiperMode)` — Sets index of current wiper speed. To get current speed or number of available speeds, check `ac.StateCar`.
- `ac.setHighBeams(active)` — Toggles high beams on and off. Note: the way lights work, this only changes if high beams are working with low beams or if low beams are alone (and, in reality, they actu
- `ac.setExtraSwitch(index, value)` — Changes the state of an extra switch. Note: if switch is in hold mode, it’ll reset to its disabled state after a few frames.  This function ignores availability checks (b
- `ac.isExtraSwitchAvailable(index, availableToCarControl)` — Checks if an extra switch is available.
- `ac.iconColor()`
- `ac.noticeNewApp(appID)` — Loads a new app if it was just added. Usually file monitoring should take care of it as well, but in some cases (like with symlinks) it might malfunction.
- `ac.uninstallApp(appID)` — Tries to unload an app and move its folder to the recycle bin. Use carefully.
- `ac.forceFadingIn()` — Forces window with “FADING” flag to fade in even if it’s not currently hovered by a cursor or focused.
- `ac.redirectVirtualMirror(active)`
- `ac.getCurrentVoteDetails()` — Returns information about the current vote, or `nil` if there is not any. Time is in seconds.
- `ac.canCastVote(voteType)` — Returns `true` if `ac.castVote()` can be called now and it’s not an offline race or a cooldown.
- `ac.castVote(voteType, vote, carIndex)` — Cast a vote online. Can be used to participate in an existing vote, or to start a new vote. Votes can only be cast once in awhile.
- `ac.restartApp()` — Try and reload app a bit later (like, in the next frame). Might cause a lag, use only in response to user actions and not in background.
- `ac.startBackgroundWorker(script, data, callback)` — Load a separate Lua script and run it in background. Script simply runs once and then the returned value (or an error message if any) will be passed to the callback. Load
- `ac.collectKN5TextureNames(kn5Filename, filter)` — Collect list of textures in KN5 file.
- `ac.collectKN5MaterialProperties(kn5Filename, filter)` — Collect list of material properties in KN5 file in a form of shader replacements config.
- `ac.Particles.Flame(params)` — Flame emitter holding specialized settings. Set settings in a table when creating an emitter and/or change them later. Use `:emit(position, velocity, amount)` to actually
- `ac.Particles.Sparks(params)` — Sparks emitter holding specialized settings. Set settings in a table when creating an emitter and/or change them later. Use `:emit(position, velocity, amount)` to actuall
- `ac.Particles.Smoke(params)` — Smoke emitter holding specialized settings. Set settings in a table when creating an emitter and/or change them later. Use `:emit(position, velocity, amount)` to actually
- `ac.Particles.Detractor(params)` — Particles detractor pushing smoke (and othe particles later) away. Move it somewhere, set radius and velocity and it would affect the smoke. Note: use carefully, smoke pa
- `ac.SeatParams(position, pitch, yaw)` — Driver seat parameters.
- `ac.getVRAMConsumption()` — Returns VRAM stats if available (older versions of Windows won’t provide access to this API). All sizes are in megabytes. Note: values provided are for a GPU in general, 
- `ac.RealMirrorParams()` — Stores Real Mirror parameters for a real view mirror.
- `ac.getRealMirrorParams(mirrorIndex)` — Returns set of mirror settings for a given Real Mirror mirror (for car scripts, associated car, for apps and tools — player’s car).
- `ac.accessExtraSwitchParams(index)`
- `ac.setColorCorrection(targetLDR, priority)` — YEBIS uses color matrices to quickly adjust HDR (input 0…∞ range) and LDR (output 0…1 range) color. Tweaks such as saturation, brightness and contrast configured in YEBIS

## 6. `render.*` — çizim (38)

- `render.isVisible(center, radius, gSpace, includeFarPlane)` — Can be called only within render context (otherwise returns `false`). Checks if given point is within current camera frustum. For checking if something is within the main
- `render.setBlendMode(mode)`
- `render.setCullMode(mode)`
- `render.setDepthMode(mode)`
- `render.glSetColor(color)`
- `render.glBegin(primitiveType)`
- `render.glVertex(v)`
- `render.glVertexTextured(v, uv)`
- `render.glTexture(filename)`
- `render.glEnd()`
- `render.circle(pos, dir, radius, color, borderColor)` — Draws a simple circle othrogonal to `dir` direction using an optimized shader.
- `render.rectangle(pos, dir, width, height, color)` — Draws a simple rectangle othrogonal to `dir` direction using an optimized shader.
- `render.quad(p1, p2, p3, p4, color, texture)` — Draws a simple quad from four points.
- `render.debugText(pos, text, color, scale, align)`
- `render.debugSphere(center, radius, color)`
- `render.debugCross(center, size, color)`
- `render.debugBox(center, size, color)`
- `render.debugPoint(center, size, color)`
- `render.debugPlane(center, dir, color, size)`
- `render.debugLine(from, to, color, colorEnd)`
- `render.debugArrow(from, to, size, color, colorEnd)`
- `render.getRenderTargetSize()`
- `render.backupRenderTarget()`
- `render.restoreRenderTarget()`
- `render.onSceneReady(callback)` — Called when shadow maps and reflection cubemap are updated, but main render has not started yet. Great time to render any scene-dependant geometry shots.
- `render.on(eventName, callback)` — Called when AC is drawing something.  Special aspects:  - `main.hud.vr`: if first argument is `true`, HUD is rendered after post-processing (or post-processing is disable
- `render.projectPoint(pos, faceID)` — Projects world point onto a screen (taking into account UV scale unless second argument is set to `false`).
- `render.createRay(pos, dir, length)` — Create a ray going from a given position along given direction.
- `render.createMouseRay()` — Create a ray going from camera in the direction of mouse position. Helpful if you want to do something like a mouse picker. Doesn’t have a length limit.
- `render.createPointRay(onscreenPoint)` — Create a ray going from camera in the direction of an onscreen point. Doesn’t have a length limit.
- `render.isPositioningHelperBusy()`
- `render.setTransform(pos, look, up, applySceneOriginOffset)` — Affects positioning of debug shapes or meshes drawn next.
- `render.bindTexture(slot, texture)` — Bind texture to a certain slot directly. If you are going to use some shader call with the same texture a lot, it might be faster to simply add texture in there with some
- `render.fullscreenPass(params)` — Draws a fullscreen pass with a custom shader. Shader is compiled at first run, which might take a few milliseconds. If you’re drawing things continuously, use `async` par
- `render.shaderedQuad(params)` — Draws a 3D quad with a custom shader. Shader is compiled at first run, which might take a few milliseconds. If you’re drawing things continuously, use `async` parameter a
- `render.mesh(params)` — Draws a 3D mesh with a custom shader. Shader is compiled at first run, which might take a few milliseconds. If you’re drawing things continuously, use `async` parameter a
- `render.measure(key, drawFn)`
- `render.PositioningHelper(params)` — A helper for moving things around using mouse with XYZ arrows.

## 7. `web.*`, `io.*`, `os.*`

- `web.timeouts(resolve, connect, send, receive)` — Configures timeouts in milliseconds for the following web requests. If you’re sure in your server, consider lowering timeouts so that in a case of a m
- `web.get(url, headers, callback)` — Sends a GET HTTP or HTTPS request. Note: you can only have two requests running at once, mostly to make sure a faulty script wouldn’t spam a remote se
- `web.post(url, headers, data, callback)` — Sends a POST HTTP or HTTPS request. Note: you can only have two requests running at once, mostly to make sure a faulty script wouldn’t spam a remote s
- `web.request(method, url, headers, data, callback)` — Sends a custom HTTP or HTTPS request. Note: you can only have two requests running at once, mostly to make sure a faulty script wouldn’t spam a remote
- `web.socket(url, headers, callback, params)` — Open a WebSocket connection.
- `web.loadRemoteModel(source, callback)` — Loads a ZIP file from a given URL, unpacks first KN5 from it to a cache folder and returns its filename through a callback. If file is already in cach
- `web.loadRemoteAnimation(source, callback)` — Loads a ZIP file from a given URL, unpacks first KsAnim from it to a cache folder and returns its filename through a callback. If file is already in c
- `web.loadRemoteAssets(source, callback)` — Loads a ZIP file from a given URL, unpacks assets from it to a cache folder and returns path to the folder in a callback. If files are already in cach
- `web.resolveDNS(url, callback)` — Resolve hostname, return IPv4 via a callback.
- `web.encryptKey(key)`
- `io.scanDrives()` — Returns list of logical drives, each drive in “A:“ format. Not available to scripts without I/O access.
- `io.getAttributes(filename)` — Gets file attributes.
- `io.getMainExecutable()` — Gets full filename of the main AC executable (“…/acs.exe” for most cases). Not available to scripts without I/O access.
- `io.load(filename, fallbackData)` — Reads file content into a string, if such file exists, otherwise returns fallback data or `nil`.
- `io.loadAsync(filename, callback)` — Reads file content into a string, if such file exists, otherwise returns fallback data or `nil`. Asynchronous version.
- `io.save(filename, data, ensure)` — Writes data into a file, returns `true` if operation was successful. Data string can contain zero bytes. Not available to scripts without I/O access.
- `io.saveAsync(filename, data, callback, ensure)` — Writes data into a file from a different thread, returns `true` via callback if operation was successful. Data string can contain zero bytes. Not avai
- `io.exists(filename)` — Checks if file or directory exists. If you need to know specifically if a file or directory exists, use `io.dirExists(filename)` or `io.fileExists(fil
- `io.dirExists(filename)` — Checks if directory exists. If there is a file in its place, it would return `false`.
- `io.fileExists(filename)` — Checks if file exists. If there is a directory in its place, it would return `false`.
- `io.fileSize(filename)` — Calculates file size in bytes. Returns -1 if there was an error.
- `io.creationTime(filename)` — Returns creation time as number of seconds since 1970, or -1 if there was an error.
- `io.lastAccessTime(filename)` — Returns last access time as number of seconds since 1970, or -1 if there was an error.
- `io.lastWriteTime(filename)` — Returns last write time as number of seconds since 1970, or -1 if there was an error.
- `io.createDir(filename)` — Creates new directory, returns `true` if directory was created. If parent directories are missing, they’ll be created as well. Not available to script
- `io.createFileDir(filename)` — Creates new directory for given filename (as in, take parent path and create directory with it). Returns `true` if directory was created. Not availabl
- `io.getParentPath(filename, level)` — Returns parent path for given filename.
- `io.getFileName(filename, noExtension)` — Returns file name path for given filename.
- `io.isFileNameAcceptable(fileName)` — Checks if file name is acceptable, returns `true` if there are no prohibited symbols in it (unlike `io.isFileNameAcceptable()`, does not allow slashes
- `io.isFilePathAcceptable(filename)` — Checks if full file name is acceptable, returns `true` if there are no prohibited symbols in it (unlike `io.isFileNameAcceptable()`, does allow slashe
- `io.move(existingFilename, newFilename, replaceExisting)` — Moves a file or a directory with all of its contents to a new place, returns `true` if moved successfully. Can be used for moving or renaming things. 
- `io.findFile(filename)` — Given an absolute or a relative path, find an actual absolute path. If script doesn’t have I/O access to such file, returns `nil`.
- `io.copyFile(existingFilename, newFilename, failIfExists)` — Copies a file to a new place, returns `true` if moved successfully. Not available to scripts without I/O access.
- `io.deleteFile(filename)` — Deletes a file, returns `true` if file was deleted successfully. To delete empty directory, use `io.deleteDir()`. If you’re operating around important
- `io.deleteDir(filename)` — Deletes an empty directory, returns `true` if directory was deleted successfully. To delete a file, use `io.deleteFile()`. Not available to scripts wi
- `io.recycle(filename)` — Moves file to Windows Recycle Bin, returns `true` if file was moved successfully. Note: this operation is much slower than removing a file with `io.de
- `io.saveToZip(zipFilename, entryFilename, data)` — Adds a new entry to a ZIP file. If there is no such ZIP file, new one will be created. If the ZIP file already has an entry with the same name, does n
- `io.loadFromZip(zipFilename, entryFilename)` — Loads file from an archive as a string. Archive would remain open for some time to speed up consequent reads. If failed, returns `nil`. Alternatively,
- `io.extractFromZipAsync(zipFilename, destination, filter, callback)` — Extract files from a ZIP file into a given directory. Creates directory if it’s missing. Not available to scripts without I/O access.
- `io.checksumSHA256(filename, callback)` — Computes SHA-256 checksum of a given file, returns result in a callback.
- `io.scanZip(zipFilename)` — Returns list of entry names from a ZIP-file.
- `io.createZipAsync(filename, entries, callback)` — Not available to scripts without I/O access.
- `io.arrtest1()` — Not available to scripts without I/O access.
- `io.arrtest2()` — Not available to scripts without I/O access.
- `io.relative(path)` — Resolves relative path to a file (relative to Lua file you’re running this function from) so it would be ready to be passed to `io` functions (returns
- `io.scanDir(directory, mask, callback, callbackData)` — Scan directory and call callback function for each of files, passing file name (not full name, but only name of the file) and attributes. If callback 
- `os.preciseClock()` — Returns time in seconds from script start (with high precision).
- `os.onURL(mask, callback, priority)` — Sets a callback which will be called when CSP itself or a CSP Lua script tries to open an URL. Return `true` if you’re handling URL, so the event woul
- `os.execute(cmd, timeoutMs, windowless)` — Altered version of regular `os.execute()`: allows to specify timeout and doesn’t show a new window.  Note: please consider using `os.runConsoleProcess
- `os.setCurrentFolder(filename)` — Changes current directory. Any argument `path`, when parsed and found not to be absolute, will be resolved against current directory. By default, uses
- `os.dateGlobal(format, timestamp)` — Returns formatted date. Same as `os.date()`, but returned value does not include system timezome.
- `os.addDLLDirectory(filename)` — Adds new directory to look for DLL files in. Warning: do not use this thing unless you really need to, and try to avoid adding LuaJIT extensions: LuaJ
- `os.showMessage(msg, type)` — Show a popup message using good old MessageBox. Please do not use it for debugging, instead consider using `ac.log()` and `ac.debug('key', 'value')` w
- `os.showInExplorer(filename)` — Shows file in Windows Explorer (opens folder with it and selects the file). Not available to scripts without I/O access.
- `os.openInExplorer(directory)` — Opens file or directory in Windows Explorer. If it’s a file, associated program will be launched instead. Not available to scripts without I/O access.
- `os.findAssociatedExecutable(filename)` — Tries to find a program associated with a filename. Returns path to it, or `nil` if nothing was found. Not available to scripts without I/O access.
- `os.openTextFile(filename, line)` — Opens text file at given line in a default text editor. Supports VS Code, Notepad++, Sublime Text and Atom (they all use different arguments for line 
- `os.openURL(url, invokeListeners)` — Opens URL in default system browser.
- `os.parseDate(date, format)` — Parse date and return a unix timestamp. Uses `std::get_time()` for actual parsing: <https://en.cppreference.com/w/cpp/io/manip/get_time>.
- `os.openFileDialog(params, callback)` — Opens regular Windows file opening dialog, calls callback with either an error or a path to a file selected by user (or nil if selection was cancelled
- `os.saveFileDialog(params, callback)` — Opens regular Windows file saving dialog, calls callback with either an error or a path to a file selected by user (or nil if selection was cancelled)
- `os.runConsoleProcess(params, callback)` — Run a console process in background with given arguments, return exit code and output in callback. Launched process will be tied to AC process to shut

## 8. `ui.*` — arayüz (389 fonksiyon)

ImGui tabanlı: pencere, buton, slider, giriş kutusu, tablo, çizim (`ui.drawLine/drawRect/drawImage...`), `ui.toast`, `ui.modalPopup`, medya oynatıcı, ekstra canvas. Tam liste için SDK dosyasına bak.
