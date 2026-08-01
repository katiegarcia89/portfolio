# Indian Creek Cycles Bug Fix Code Comparison

This document compares the original committed code against the current changed code.

- Lines starting with `-` are from the original code before the bug fixes.
- Lines starting with `+` are the changed code currently used by the app.
- Inline comments in the changed code explain which QA bug each fix addresses and why the fix exists.


## Changed Source Files

diff --git a/config/settings.py b/config/settings.py
index ff017cb1..4d8f71d9 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -125,6 +125,8 @@ OPENWEATHER_API_KEY = os.getenv('OPENWEATHER_API_KEY', '')
 # Session settings
 SESSION_COOKIE_AGE = 86400  # 24 hours
 SESSION_EXPIRE_AT_BROWSER_CLOSE = False
+# BUG-002/BUG-003: give pending reservations a short grace window before auto-cancelling them.
+PENDING_RESERVATION_GRACE_HOURS = int(os.getenv('PENDING_RESERVATION_GRACE_HOURS', '24'))
 
 # Message tags
 from django.contrib.messages import constants as messages
diff --git a/core/views.py b/core/views.py
index 90ac819d..d32ab9b4 100644
--- a/core/views.py
+++ b/core/views.py
@@ -24,6 +24,7 @@ from bikes.models import Accessory, Bike, BikeCategory
 from reviews.models import Review
 from reviews.forms import AdminReviewCommentForm
 from reservations.models import PromoCode, Reservation, ReservationAccessory, Waiver
+from reservations.utils import expire_pending_reservations
 from payments.models import Payment
 from .models import Trail
 from .forms import (
@@ -244,10 +245,16 @@ User = get_user_model()
 
 @staff_member_required
 def admin_dashboard(request):
+    # BUG-002/BUG-003: clear stale pending holds before dashboard totals are calculated.
+    expire_pending_reservations()
+    today = timezone.localdate()
     recent_reservations = Reservation.objects.select_related("user", "bike").order_by("-created_at")[:8]
     total_reservations = Reservation.objects.count()
+    # BUG-001: active means happening today, not every pending or future reservation.
     active_reservations = Reservation.objects.filter(
-        status__in=["pending", "confirmed", "paid", "active"]
+        status__in=["confirmed", "paid", "active"],
+        rental_date__lte=today,
+        return_date__gte=today,
     ).count()
 
     total_bikes = Bike.objects.count()
@@ -265,8 +272,11 @@ def admin_dashboard(request):
     total_reviews = Review.objects.count()
     pending_reviews = Review.objects.filter(is_approved=False).count()
     signed_waivers = Waiver.objects.count()
+    # BUG-001: waiver follow-up should only include currently active paid/confirmed rides.
     unsigned_active_waivers = Reservation.objects.filter(
-        status__in=["pending", "confirmed", "paid", "active"],
+        status__in=["confirmed", "paid", "active"],
+        rental_date__lte=today,
+        return_date__gte=today,
         waiver_signed=False,
     ).count()
     user_count = User.objects.count()
@@ -391,12 +401,19 @@ def admin_dashboard(request):
         or Decimal("0.00")
     )
     merchandise_revenue = Decimal("0.00")
+    rental_addon_revenue = Decimal("0.00")
+    # BUG-004: rental add-ons are revenue too, so include them in the dashboard total.
+    for item in ReservationAccessory.objects.filter(
+        fulfillment_type="rental",
+        reservation__status__in=["paid", "active", "completed"],
+    ):
+        rental_addon_revenue += item.get_total()
     for item in ReservationAccessory.objects.filter(
         fulfillment_type="purchase",
         reservation__status__in=["paid", "active", "completed"],
     ):
         merchandise_revenue += item.get_total()
-    total_revenue = rental_revenue + merchandise_revenue
+    total_revenue = rental_revenue + rental_addon_revenue + merchandise_revenue
 
     context = {
         "total_bikes": total_bikes,
@@ -407,6 +424,7 @@ def admin_dashboard(request):
         "total_payments": total_payments,
         "completed_payments": completed_payments,
         "rental_revenue": rental_revenue,
+        "rental_addon_revenue": rental_addon_revenue,
         "merchandise_revenue": merchandise_revenue,
         "total_revenue": total_revenue,
         "total_reviews": total_reviews,
@@ -953,6 +971,7 @@ def toggle_bike_maintenance(request, bike_id):
 
 @staff_member_required
 def admin_reservations(request):
+    expire_pending_reservations()
     reservations = Reservation.objects.select_related("user", "bike").order_by("-created_at")
     return render(request, "admin_dashboard/admin_reservations.html", {"reservations": reservations})
 
@@ -1149,6 +1168,7 @@ def void_payment(request, payment_id):
 
 @staff_member_required
 def admin_waivers(request):
+    expire_pending_reservations()
     signed_waivers = Waiver.objects.select_related(
         "user",
         "reservation",
@@ -1158,7 +1178,9 @@ def admin_waivers(request):
         "user",
         "bike",
     ).filter(
-        status__in=["pending", "confirmed", "paid", "active"],
+        status__in=["confirmed", "paid", "active"],
+        rental_date__lte=timezone.localdate(),
+        return_date__gte=timezone.localdate(),
         waiver_signed=False,
     ).order_by("rental_date", "created_at")
 
diff --git a/reservations/utils.py b/reservations/utils.py
index 623bf09a..42fba778 100644
--- a/reservations/utils.py
+++ b/reservations/utils.py
@@ -1,13 +1,28 @@
-# reservations/utils.py
+from datetime import timedelta
+from io import BytesIO
+import base64
 
 from django.core.mail import EmailMultiAlternatives
 from django.template.loader import render_to_string
 from django.utils.html import strip_tags
 from django.conf import settings
 from django.urls import reverse
+from django.utils import timezone
 import qrcode
-from io import BytesIO
-import base64
+
+from .models import Reservation
+
+
+def expire_pending_reservations(now=None):
+    """Cancel stale pending reservations so they no longer appear as active holds."""
+    now = now or timezone.now()
+    grace_hours = getattr(settings, 'PENDING_RESERVATION_GRACE_HOURS', 24)
+    expiration_date = (timezone.localtime(now) - timedelta(hours=grace_hours)).date()
+
+    return Reservation.objects.filter(
+        status='pending',
+        rental_date__lte=expiration_date,
+    ).update(status='cancelled')
 
 def send_confirmation_email(reservation):
     subject = f"Reservation Confirmed - {reservation.bike.name}"
@@ -39,4 +54,4 @@ def send_confirmation_email(reservation):
     )
 
     email.attach_alternative(html_content, "text/html")
-    email.send()
\ No newline at end of file
+    email.send()
diff --git a/static/css/main.css b/static/css/main.css
index 37112815..f504dba8 100644
--- a/static/css/main.css
+++ b/static/css/main.css
@@ -385,6 +385,13 @@ img {
     transform: rotate(180deg);
 }
 
+/* BUG-005: at high browser zoom, the logo already acts as Home, so hide the duplicate link before it overlaps the logo. */
+@media (max-width: 1240px) and (min-width: 701px) {
+    .navbar-nav > .nav-item:first-child {
+        display: none;
+    }
+}
+
 /* ============================================
    Messages / Alerts
    ============================================ */
@@ -2343,7 +2350,7 @@ textarea.form-control {
 /* ============================================
    Responsive
    ============================================ */
-@media (max-width: 991px) {
+@media (max-width: 700px) {
     .navbar-toggle {
         display: flex;
         order: 3;
diff --git a/static/js/main.js b/static/js/main.js
index c6318faa..0f7b1953 100644
--- a/static/js/main.js
+++ b/static/js/main.js
@@ -592,4 +592,4 @@ function getCSRFToken() {
     return document.cookie.split('; ')
         .find(row => row.startsWith('csrftoken='))
         ?.split('=')[1];
-}
\ No newline at end of file
+}
diff --git a/staticfiles/css/main.css b/staticfiles/css/main.css
index 37112815..f504dba8 100644
--- a/staticfiles/css/main.css
+++ b/staticfiles/css/main.css
@@ -385,6 +385,13 @@ img {
     transform: rotate(180deg);
 }
 
+/* BUG-005: at high browser zoom, the logo already acts as Home, so hide the duplicate link before it overlaps the logo. */
+@media (max-width: 1240px) and (min-width: 701px) {
+    .navbar-nav > .nav-item:first-child {
+        display: none;
+    }
+}
+
 /* ============================================
    Messages / Alerts
    ============================================ */
@@ -2343,7 +2350,7 @@ textarea.form-control {
 /* ============================================
    Responsive
    ============================================ */
-@media (max-width: 991px) {
+@media (max-width: 700px) {
     .navbar-toggle {
         display: flex;
         order: 3;
diff --git a/staticfiles/js/main.js b/staticfiles/js/main.js
index a5ae308f..d9582f6d 100644
--- a/staticfiles/js/main.js
+++ b/staticfiles/js/main.js
@@ -475,4 +475,4 @@ document.addEventListener('DOMContentLoaded', function () {
             onChange: validateTimes
         });
     }
-}); 
\ No newline at end of file
+}); 
diff --git a/templates/admin_dashboard/admin.html b/templates/admin_dashboard/admin.html
index 96c4c48f..238343a7 100644
--- a/templates/admin_dashboard/admin.html
+++ b/templates/admin_dashboard/admin.html
@@ -410,7 +410,9 @@
                     <div class="stat-value">${{ total_revenue|floatformat:0 }}</div>
                 </div>
                 <div class="stat-meta">
+                    <!-- BUG-004: show the same revenue parts used in the total so the dashboard matches the breakdown page. -->
                     <span class="stat-chip"><strong>${{ rental_revenue|floatformat:0 }}</strong> rentals</span>
+                    <span class="stat-chip"><strong>${{ rental_addon_revenue|floatformat:0 }}</strong> add-ons</span>
                     <span class="stat-chip"><strong>${{ merchandise_revenue|floatformat:0 }}</strong> merchandise</span>
                 </div>
             </div>
diff --git a/templates/base.html b/templates/base.html
index a66e56f4..7497cf7f 100644
--- a/templates/base.html
+++ b/templates/base.html
@@ -13,7 +13,7 @@
     <link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@400;500;600;700&family=Inter:wght@300;400;500;600&display=swap" rel="stylesheet">
     
     <!-- CSS -->
-    <link rel="stylesheet" href="{% static 'css/main.css' %}">
+    <link rel="stylesheet" href="{% static 'css/main.css' %}?v=live-nav-hide-home">
     <link rel="stylesheet" href="{% static 'css/reservations.css' %}">
     <link rel="stylesheet" href="{% static 'css/profile.css' %}">
     <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
@@ -239,7 +239,7 @@
     
      <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/flatpickr/dist/flatpickr.min.css">
     <script src="https://cdn.jsdelivr.net/npm/flatpickr"></script>
-    <script src="{% static 'js/main.js' %}"></script>
+    <script src="{% static 'js/main.js' %}?v=live-nav-hide-home"></script>
     {% block extra_js %}{% endblock %}
 </body>
 </html>
diff --git a/templates/reservations/help.html b/templates/reservations/help.html
index bc73c348..78b57cf4 100644
--- a/templates/reservations/help.html
+++ b/templates/reservations/help.html
@@ -246,6 +246,17 @@
             </div>
         </div>
 
+        <!-- BUG-002/BUG-003: explain when old pending reservations are released back to inventory. -->
+        <div class="faq-item">
+            <button class="faq-question">
+                How long will a pending reservation be held?
+                <span class="faq-icon"><i class="fas fa-chevron-down"></i></span>
+            </button>
+            <div class="faq-answer">
+                <p>Pending reservations are held through the scheduled start date. If payment or staff confirmation has not been completed within 24 hours after that start date, the reservation may be automatically cancelled so the bike can be made available again. Contact us before that window closes if you need an exception.</p>
+            </div>
+        </div>
+
         <div class="faq-item">
             <button class="faq-question">
                 What if it rains on my rental day?
@@ -311,7 +322,9 @@
 
 </div>
 <script>
+// BUG-006: keep FAQ expanded/collapsed state available to keyboard and screen-reader users.
 document.querySelectorAll(".faq-question").forEach(button => {
+    button.setAttribute("aria-expanded", "false");
     button.addEventListener("click", () => {
         const faqItem = button.parentElement;
 
@@ -319,11 +332,16 @@ document.querySelectorAll(".faq-question").forEach(button => {
         document.querySelectorAll(".faq-item").forEach(item => {
             if (item !== faqItem) {
                 item.classList.remove("active");
+                const itemButton = item.querySelector(".faq-question");
+                if (itemButton) {
+                    itemButton.setAttribute("aria-expanded", "false");
+                }
             }
         });
 
         // Toggle current
         faqItem.classList.toggle("active");
+        button.setAttribute("aria-expanded", faqItem.classList.contains("active") ? "true" : "false");
     });
 });
 </script>

## Database Note

`db.sqlite3` also changed during local testing/data updates, but it is binary and not shown as code here. The original committed database backup is saved as `db_original.sqlite3`.
