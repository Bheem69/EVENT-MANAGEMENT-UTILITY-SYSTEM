# Code Differences Report: newGood vs EVENT-MANAGEMENT-UTILITY-SYSTEM

**Generated:** April 17, 2026  
**Comparison:** newGood (source) → EVENT-MANAGEMENT-UTILITY-SYSTEM/Branch1 (target)

---

## Executive Summary

**Total Files Changed:** 13 files  
**Status:** ❌ NOT IDENTICAL - Multiple differences found

### Categories of Changes:
1. **Security Features** - Admin secret key validation removed
2. **Authorization Logic** - Event ownership checks removed
3. **Database Schema** - Event status field removed
4. **Configuration** - Database credentials and URLs differ
5. **Dependencies** - Validation dependency removed
6. **UI Features** - Event status display removed

---

## 1. BACKEND CHANGES

### 1.1 EventManagementApplication.java
**Location:** `src/main/java/com/example/eventmanagement/EventManagementApplication.java`

**Lines Changed:** 1-25

**Changes:**
- ❌ **MISSING in EVENT-MANAGEMENT-UTILITY-SYSTEM:** Admin promotion CommandLineRunner (lines 19-36 in newGood)
- ❌ **MISSING imports:** UserRepository, @Value, CommandLineRunner, @Bean

**Impact:** Cannot auto-promote users to admin role on startup

**newGood Code (lines 19-36):**
```java
@Bean
CommandLineRunner promoteAdmin(
        UserRepository userRepository,
        @Value("${app.init-admin-username:}") String adminUsername) {
    return args -> {
        if (adminUsername.isBlank()) return;
        userRepository.findByUsername(adminUsername).ifPresent(user -> {
            if (!"ROLE_ADMIN".equals(user.getRole())) {
                user.setRole("ROLE_ADMIN");
                userRepository.save(user);
                System.out.println("[Init] Promoted '" + adminUsername + "' to ROLE_ADMIN");
            } else {
                System.out.println("[Init] '" + adminUsername + "' is already ROLE_ADMIN");
            }
        });
    };
}
```

**EVENT-MANAGEMENT-UTILITY-SYSTEM:** This entire method is MISSING

---

### 1.2 SecurityConfig.java
**Location:** `src/main/java/com/example/eventmanagement/config/SecurityConfig.java`

**Lines Changed:** 41

**Changes:**
- **Line 41 in newGood:** `.requestMatchers("/", "/login", "/register", "/error").permitAll()`
- **Line 41 in EVENT-MANAGEMENT-UTILITY-SYSTEM:** `.requestMatchers("/login", "/register").permitAll()`

**Impact:** Root path "/" and "/error" not publicly accessible in EVENT-MANAGEMENT-UTILITY-SYSTEM

---

### 1.3 AuthController.java
**Location:** `src/main/java/com/example/eventmanagement/controller/AuthController.java`

**Lines Changed:** 17-60

**Changes:**

#### ❌ MISSING: Admin Secret Validation (lines 20-23, 48-58 in newGood)
```java
// Lines 20-23 - MISSING in EVENT-MANAGEMENT-UTILITY-SYSTEM
@Value("${app.admin-secret:admin123}")
private String adminSecret;

// Lines 48-58 - MISSING in EVENT-MANAGEMENT-UTILITY-SYSTEM
if ("admin".equals(registerRole)) {
    if (secret == null || !adminSecret.equals(secret.trim())) {
        User u = new User();
        u.setUsername(username);
        u.setEmail(email);
        model.addAttribute("user", u);
        model.addAttribute("selectedRole", "admin");
        model.addAttribute("error", "Invalid admin secret key.");
        return "register";
    }
}
```

#### ❌ MISSING: Root redirect endpoint (lines 26-29 in newGood)
```java
@GetMapping("/")
public String redirectToLogin() {
    return "redirect:/login";
}
```

**Impact:** No admin secret validation - anyone can register as admin

---

### 1.4 EventController.java
**Location:** `src/main/java/com/example/eventmanagement/controller/EventController.java`

**Lines Changed:** Multiple sections (19-22, 78-98, 114-223)

**Major Changes:**

#### ❌ REMOVED: Base URL configuration (lines 22-23 in newGood)
```java
@Value("${app.base-url:http://localhost:8080}")
private String baseUrl;
```

#### ❌ CHANGED: QR Code generation (lines 85-88)
**newGood:**
```java
String scanUrl = baseUrl + "/admin/scan?ticketCode=" + booking.getTicketCode();
String qrBase64 = qrCodeService.generateQrBase64(scanUrl, 280, 280);
```

**EVENT-MANAGEMENT-UTILITY-SYSTEM:**
```java
String baseUrl = request.getScheme() + "://" + request.getServerName()
        + (request.getServerPort() != 80 && request.getServerPort() != 443
                    ? ":" + request.getServerPort() : "");
String scanUrl = baseUrl + "/admin/scan?ticketCode=" + booking.getTicketCode();
String qrBase64 = qrCodeService.generateQrBase64(scanUrl, 250, 250);
```

#### ❌ REMOVED: Event ownership checks in multiple methods

**adminEventList() - Line 98-100:**
- **newGood:** `eventService.getEventsByCreator(userDetails.getUsername())`
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** `eventService.getAllEvents()` (shows ALL events, not just user's)

**createEvent() - Lines 112-125:**
- **newGood:** Has try-catch with error handling
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** No error handling

**editEventForm() - Lines 127-142:**
- **newGood:** Checks if user owns the event (lines 131-134)
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** No ownership check

**updateEvent() - Lines 144-157:**
- **newGood:** Validates ownership before update
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** No validation

**deleteEvent() - Lines 159-169:**
- **newGood:** Validates ownership before delete
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** No validation

**eventBookings() - Lines 171-186:**
- **newGood:** Checks if user owns event before showing bookings
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** No check

**scanPage() & processTicket() - Lines 191-223:**
- **newGood:** Passes `userDetails.getUsername()` to validate admin owns the event
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** No username passed, no ownership check

**Impact:** Any admin can edit/delete ANY event, not just their own

---

### 1.5 EventService.java
**Location:** `src/main/java/com/example/eventmanagement/service/EventService.java`

**Lines Changed:** 26-66

**Changes:**

#### ❌ REMOVED: getEventsByCreator() method (lines 26-28 in newGood)
```java
public List<Event> getEventsByCreator(String username) {
    return eventRepository.findByCreatedBy(username);
}
```

#### ❌ REMOVED: Duplicate title check in createEvent() (lines 31-34 in newGood)
```java
if (eventRepository.existsByTitleAndCreatedBy(event.getTitle(), event.getCreatedBy())) {
    throw new RuntimeException("You already have an event with this title. Please use a different title.");
}
```

#### ❌ REMOVED: Ownership validation in updateEvent() (lines 38-41 in newGood)
```java
if (!existing.getCreatedBy().equals(username)) {
    throw new RuntimeException("You can only edit your own events.");
}
```

#### ❌ REMOVED: Ownership validation in deleteEvent() (lines 58-61 in newGood)
```java
Event event = getEventById(id);
if (!event.getCreatedBy().equals(username)) {
    throw new RuntimeException("You can only delete your own events.");
}
```

**Impact:** No service-level authorization checks

---

### 1.6 BookingService.java
**Location:** `src/main/java/com/example/eventmanagement/service/BookingService.java`

**Lines Changed:** 86-95

**Changes:**

#### ❌ REMOVED: Admin ownership check for ticket scanning (lines 89-92 in newGood)
```java
// Check if the admin is the creator of this event
if (!booking.getEvent().getCreatedBy().equals(adminUsername)) {
    throw new RuntimeException("You can only scan tickets for your own events.");
}
```

**Method signature changed:**
- **newGood:** `validateAndUseTicket(String ticketCode, String adminUsername)`
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** `validateAndUseTicket(String ticketCode)`

**Impact:** Any admin can scan tickets for any event

---

### 1.7 TicketEmailService.java
**Location:** `src/main/java/com/example/eventmanagement/service/TicketEmailService.java`

**Lines Changed:** 27

**Changes:**
- **newGood Line 27:** Has comment: `// Set to false if mail is not configured — app still works, just no email`
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** Comment removed

**Impact:** Minor documentation change only

---

### 1.8 Event.java (Model)
**Location:** `src/main/java/com/example/eventmanagement/model/Event.java`

**Lines Changed:** 7-8, 36-39, 60-62

**Changes:**

#### ❌ REMOVED: Unique constraint on title + createdBy (lines 7-8 in newGood)
**newGood:**
```java
@Table(name = "events", 
       uniqueConstraints = @UniqueConstraint(columnNames = {"title", "createdBy"}))
```

**EVENT-MANAGEMENT-UTILITY-SYSTEM:**
```java
@Table(name = "events")
```

#### ❌ REMOVED: Status field (lines 36-37 in newGood)
```java
@Column(nullable = false)
private String status = "PUBLISHED"; // DRAFT, PUBLISHED, CANCELLED, COMPLETED
```

#### ❌ REMOVED: Status getter/setter (lines 60-62 in newGood)
```java
public String getStatus() { return status; }
public void setStatus(String status) { this.status = status; }
```

**Impact:** 
- Multiple events can have same title from same creator
- No event status tracking (draft, published, cancelled, completed)

---

### 1.9 EventRepository.java
**Location:** `src/main/java/com/example/eventmanagement/repository/EventRepository.java`

**Lines Changed:** 10-11

**Changes:**

#### ❌ REMOVED: Two repository methods (lines 10-11 in newGood)
```java
boolean existsByTitleAndCreatedBy(String title, String createdBy);
List<Event> findByCreatedBy(String createdBy);
```

**Impact:** Cannot query events by creator or check for duplicate titles per user

---

## 2. FRONTEND CHANGES

### 2.1 register.html
**Location:** `src/main/resources/templates/register.html`

**Lines Changed:** 90-98, 144-152

**Changes:**

#### ❌ REMOVED: Admin secret key input field (lines 90-98 in newGood)
```html
<div class="mb-3" id="secretField" style="display:none;">
    <label class="form-label fw-semibold" style="font-size:.9rem;">Admin Secret Key</label>
    <div class="input-group">
        <span class="input-group-text"><i class="bi bi-key-fill"></i></span>
        <input type="password" name="secret" id="secretInput" class="form-control"
               placeholder="Enter admin secret key"/>
    </div>
</div>
```

#### ❌ REMOVED: JavaScript for secret field toggle (lines 144-146, 149 in newGood)
```javascript
const secretField = document.getElementById('secretField');
const secretInput = document.getElementById('secretInput');
// ...
secretField.style.display = isAdmin ? 'block' : 'none';
secretInput.required = isAdmin;
```

#### ❌ CHANGED: Form subtitle text (line 151 in newGood)
- **newGood:** `'Register a new administrator'`
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** `'Register as event organizer'`

**Impact:** No UI for admin secret key entry

---

### 2.2 admin/event-form.html
**Location:** `src/main/resources/templates/admin/event-form.html`

**Lines Changed:** 115, 125-135

**Changes:**

#### ❌ CHANGED: Currency symbol (line 125)
- **newGood:** `₹` (Indian Rupee)
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** `$` (USD)

#### ❌ REMOVED: Event status dropdown (lines 131-139 in newGood)
```html
<div class="mb-4" th:if="${event.id != null}">
    <label class="form-label">Event Status</label>
    <select class="form-select" th:field="*{status}">
        <option value="PUBLISHED">Published (Visible to users)</option>
        <option value="DRAFT">Draft (Hidden from users)</option>
        <option value="CANCELLED">Cancelled</option>
        <option value="COMPLETED">Completed</option>
    </select>
</div>
```

**Impact:** Cannot set event status from UI

---

### 2.3 admin/event-management.html
**Location:** `src/main/resources/templates/admin/event-management.html`

**Lines Changed:** 61-68, 104, 127-132

**Changes:**

#### ❌ REMOVED: Status badge CSS (lines 61-68 in newGood)
```css
.status-badge {
    border-radius: 20px; padding: .2rem .75rem;
    font-size: .75rem; font-weight: 600;
    text-transform: uppercase;
}
.status-published { background: #dcfce7; color: #16a34a; }
.status-draft { background: #f3f4f6; color: #6b7280; }
.status-cancelled { background: #fee2e2; color: #dc2626; }
.status-completed { background: #dbeafe; color: #2563eb; }
```

#### ❌ REMOVED: Status column header (line 104 in newGood)
```html
<th>Status</th>
```

#### ❌ REMOVED: Status badge display (lines 127-132 in newGood)
```html
<td>
    <span class="status-badge"
          th:classappend="${'status-' + #strings.toLowerCase(event.status ?: 'published')}"
          th:text="${event.status ?: 'PUBLISHED'}"></span>
</td>
```

**Impact:** No visual indication of event status

---

## 3. CONFIGURATION CHANGES

### 3.1 pom.xml
**Location:** `pom.xml`

**Lines Changed:** 95-100

**Changes:**

#### ❌ REMOVED: Validation dependency (lines 95-100 in newGood)
```xml
<!-- Validation -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

**Impact:** No Bean Validation support (if needed)

---

### 3.2 application.properties
**Location:** `src/main/resources/application.properties`

**Lines Changed:** 4, 19, 24-27, 30-31

**Changes:**

#### ⚠️ DIFFERENT: Database password (line 4)
- **newGood:** `spring.datasource.password=pranjal2005`
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** `spring.datasource.password=root12345`

#### ❌ REMOVED: Server address comment (lines 19-20 in newGood)
```properties
# server.address=0.0.0.0 
#comment this if using local host
```

#### ⚠️ DIFFERENT: Base URL (line 24)
- **newGood:** `app.base-url=http://localhost:8080`
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:** `app.base-url=http://192.168.1.6:8080`

#### ❌ MISSING: Admin secret configuration (lines 26-27 in newGood)
```properties
# Secret key to access /register/admin?secret=... — change this before deploying
app.admin-secret=11
```

#### ❌ MISSING: Init admin username (lines 29-30 in newGood)
```properties
# One-time admin promotion — set to your username, restart app, then remove this line
app.init-admin-username=qw
```

#### ⚠️ DIFFERENT: Email configuration (lines 30-31)
- **newGood:** 
  ```properties
  spring.mail.username=enter Your Email
  spring.mail.password=Enter App Password(auto-generated wala)
  ```
- **EVENT-MANAGEMENT-UTILITY-SYSTEM:**
  ```properties
  spring.mail.username=chandrahul1001@gmail.com
  spring.mail.password=12345
  ```

#### ❌ MOVED: Comment position (line 36)
- Comment about App Password moved to different line

**Impact:** Different runtime configuration, missing admin features

---

## 4. SUMMARY OF CRITICAL DIFFERENCES

### Security Issues in EVENT-MANAGEMENT-UTILITY-SYSTEM:
1. ❌ **No admin secret validation** - Anyone can register as admin
2. ❌ **No event ownership checks** - Any admin can edit/delete any event
3. ❌ **No ticket scanning authorization** - Any admin can scan any event's tickets
4. ❌ **Shows all events to all admins** - No isolation between event creators

### Missing Features in EVENT-MANAGEMENT-UTILITY-SYSTEM:
1. ❌ Event status tracking (DRAFT, PUBLISHED, CANCELLED, COMPLETED)
2. ❌ Duplicate event title prevention per user
3. ❌ Auto-admin promotion on startup
4. ❌ Root path "/" redirect
5. ❌ Validation dependency

### Configuration Differences:
1. ⚠️ Different database password
2. ⚠️ Different base URL (localhost vs IP)
3. ⚠️ Different email credentials
4. ❌ Missing admin secret config
5. ❌ Missing init admin username config

---

## 5. RECOMMENDATION

**Status:** ❌ **NOT PRODUCTION READY**

The EVENT-MANAGEMENT-UTILITY-SYSTEM version has **significant security vulnerabilities** due to removed authorization checks. 

### To make them identical, you need to:
1. Copy all authorization logic from newGood
2. Add back the Event status field and related code
3. Restore admin secret validation
4. Add back repository methods for event ownership queries
5. Update configuration files with proper values
6. Add validation dependency back to pom.xml

### Files that need to be copied from newGood:
- EventManagementApplication.java
- SecurityConfig.java
- AuthController.java
- EventController.java
- EventService.java
- BookingService.java
- Event.java
- EventRepository.java
- register.html
- admin/event-form.html
- admin/event-management.html
- pom.xml
- application.properties (with your specific credentials)

---

**Report End**
