package com.example.smartemergencymedicalasistance;

import android.Manifest;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.location.Location;
import android.net.ConnectivityManager;
import android.net.NetworkInfo;
import android.net.Uri;
import android.os.Build;
import android.os.Bundle;
import android.os.Handler;
import android.provider.Settings;
import android.telephony.SmsManager;
import android.widget.Button;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;
import androidx.core.app.NotificationCompat;

import com.google.android.gms.location.*;
import com.google.firebase.auth.FirebaseAuth;
import com.google.firebase.firestore.*;

import java.util.ArrayList;
import java.util.List;

public class MainActivity extends AppCompatActivity {

    private static final int PERMISSION_REQUEST_CODE = 100;
    private static final String CHANNEL_ID = "sos_notification";

    private Button btnSOS, btnProfile;
    private FusedLocationProviderClient fusedLocationClient;
    private LocationCallback locationCallback;

    private double userLatitude = 0.0;
    private double userLongitude = 0.0;

    private boolean locationReceived = false;
    private boolean callMade = false;

    private FirebaseFirestore db;
    private FirebaseAuth auth;

    private final List<String> emergencyContacts = new ArrayList<>();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        btnSOS = findViewById(R.id.btnSOS);
        btnProfile = findViewById(R.id.btnProfile);

        fusedLocationClient = LocationServices.getFusedLocationProviderClient(this);
        db = FirebaseFirestore.getInstance();
        auth = FirebaseAuth.getInstance();

        if (auth.getCurrentUser() == null) {
            startActivity(new Intent(this, LoginActivity.class));
            finish();
            return;
        }

        createNotificationChannel();

        checkAndRequestPermissions();

        btnSOS.setOnClickListener(v -> {
            // Prevent rapid multiple presses
            if (callMade) {
                Toast.makeText(this, "SOS already in progress. Please wait.", Toast.LENGTH_SHORT).show();
                return;
            }

            if (emergencyContacts.isEmpty()) {
                Toast.makeText(this, "Emergency contacts not loaded yet. Please check your profile.", Toast.LENGTH_LONG).show();
                loadEmergencyContacts(); // Try to reload contacts
                return;
            }

            handleSOSButtonClick();
        });

        btnProfile.setOnClickListener(v -> startActivity(new Intent(MainActivity.this, ProfileActivity.class)));
    }

    private void checkAndRequestPermissions() {
        List<String> permissionsNeeded = new ArrayList<>();

        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED)
            permissionsNeeded.add(Manifest.permission.ACCESS_FINE_LOCATION);
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.SEND_SMS) != PackageManager.PERMISSION_GRANTED)
            permissionsNeeded.add(Manifest.permission.SEND_SMS);
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.CALL_PHONE) != PackageManager.PERMISSION_GRANTED)
            permissionsNeeded.add(Manifest.permission.CALL_PHONE);

        if (!permissionsNeeded.isEmpty()) {
            boolean showRationale = false;
            for (String perm : permissionsNeeded) {
                if (ActivityCompat.shouldShowRequestPermissionRationale(this, perm)) {
                    showRationale = true;
                    break;
                }
            }

            if (showRationale) {
                new AlertDialog.Builder(this)
                        .setTitle("Permissions Needed")
                        .setMessage("This app requires location, SMS, and phone call permissions to function correctly for emergencies.")
                        .setPositiveButton("Grant", (dialog, which) ->
                                ActivityCompat.requestPermissions(this, permissionsNeeded.toArray(new String[0]), PERMISSION_REQUEST_CODE))
                        .setNegativeButton("Cancel", (dialog, which) ->
                                Toast.makeText(this, "Permissions denied. Limited functionality.", Toast.LENGTH_LONG).show())
                        .show();
            } else {
                ActivityCompat.requestPermissions(this, permissionsNeeded.toArray(new String[0]), PERMISSION_REQUEST_CODE);
            }
        } else {
            startLocationAndLoadContacts();
        }
    }

    private void startLocationAndLoadContacts() {
        requestLocationUpdates();
        loadEmergencyContacts();
    }

    private void requestLocationUpdates() {
        LocationRequest locationRequest = LocationRequest.create()
                .setInterval(3000)
                .setFastestInterval(2000)
                .setPriority(Priority.PRIORITY_HIGH_ACCURACY);

        locationCallback = new LocationCallback() {
            @Override
            public void onLocationResult(@NonNull LocationResult locationResult) {
                if (locationResult == null) return;

                Location location = locationResult.getLastLocation();
                if (location != null) {
                    userLatitude = location.getLatitude();
                    userLongitude = location.getLongitude();
                    locationReceived = true;
                    fusedLocationClient.removeLocationUpdates(locationCallback);
                }
            }
        };

        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) == PackageManager.PERMISSION_GRANTED) {
            fusedLocationClient.requestLocationUpdates(locationRequest, locationCallback, null);
        } else {
            Toast.makeText(this, "Location permission not granted.", Toast.LENGTH_SHORT).show();
        }
    }

    private void loadEmergencyContacts() {
        String userId = auth.getCurrentUser() != null ? auth.getCurrentUser().getUid() : null;
        if (userId == null) return;

        db.collection("users").document(userId)
                .get()
                .addOnSuccessListener(doc -> {
                    List<String> contacts = (List<String>) doc.get("emergencyContacts");
                    emergencyContacts.clear();
                    if (contacts != null) {
                        emergencyContacts.addAll(contacts);
                    } else {
                        Toast.makeText(this, "No emergency contacts found. Please add in profile.", Toast.LENGTH_LONG).show();
                    }
                })
                .addOnFailureListener(e -> Toast.makeText(this, "Failed to load emergency contacts.", Toast.LENGTH_SHORT).show());
    }

    private void handleSOSButtonClick() {
        callMade = true;
        locationReceived = false;

        Toast.makeText(this, "Acquiring location...", Toast.LENGTH_SHORT).show();
        requestLocationUpdates();

        // Wait max 5 seconds for location
        new Handler().postDelayed(() -> {
            if (!locationReceived) {
                Toast.makeText(this, "Failed to get location. Please try again.", Toast.LENGTH_SHORT).show();
                callMade = false;
                return;
            }

            if (!isInternetAvailable()) {
                // Send SMS with location link if no internet
                sendSmsToEmergencyContacts("🚨 SOS Alert! Internet unavailable. Last known location: " +
                        "https://maps.google.com/?q=" + userLatitude + "," + userLongitude);
                showSOSNotification("SOS sent via SMS (no internet).");
                callMade = false;
                return;
            }

            // Internet available: find nearest hospital and notify
            findNearestHospitalAndNotify();

        }, 5000); // increased wait to 5 seconds to improve location accuracy
    }

    private void findNearestHospitalAndNotify() {
        String userId = auth.getCurrentUser() != null ? auth.getCurrentUser().getUid() : null;
        if (userId == null) {
            Toast.makeText(this, "User not authenticated.", Toast.LENGTH_SHORT).show();
            callMade = false;
            return;
        }

        db.collection("users").document(userId).collection("hospitals")
                .get()
                .addOnSuccessListener(query -> {
                    if (query.isEmpty()) {
                        Toast.makeText(this, "No registered hospitals found nearby.", Toast.LENGTH_SHORT).show();
                        callMade = false;
                        return;
                    }

                    DocumentSnapshot nearest = null;
                    double minDist = Double.MAX_VALUE;

                    for (DocumentSnapshot doc : query) {
                        Double lat = doc.getDouble("latitude");
                        Double lon = doc.getDouble("longitude");
                        if (lat == null || lon == null) continue;

                        double dist = distance(userLatitude, userLongitude, lat, lon);
                        if (dist < minDist) {
                            minDist = dist;
                            nearest = doc;
                        }
                    }

                    if (nearest == null) {
                        Toast.makeText(this, "Unable to determine nearest hospital.", Toast.LENGTH_SHORT).show();
                        callMade = false;
                        return;
                    }

                    String name = nearest.getString("name") != null ? nearest.getString("name") : "Hospital";
                    String phone = nearest.getString("phoneNumber");

                    if (phone == null || phone.trim().isEmpty()) {
                        Toast.makeText(this, "Hospital phone number missing.", Toast.LENGTH_SHORT).show();
                        callMade = false;
                        return;
                    }

                    makePhoneCall(phone);

                    String message = "🚨 SOS Alert!\nNearest Hospital: " + name +
                            "\nLocation: https://maps.google.com/?q=" + userLatitude + "," + userLongitude;

                    sendSmsToEmergencyContacts(message);

                    showSOSNotification("SOS sent to " + name);
                    Toast.makeText(this, "Calling " + name, Toast.LENGTH_LONG).show();
                    callMade = false;
                })
                .addOnFailureListener(e -> {
                    Toast.makeText(this, "Failed to fetch hospital data.", Toast.LENGTH_SHORT).show();
                    callMade = false;
                });
    }

    private void makePhoneCall(String phoneNumber) {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.CALL_PHONE) == PackageManager.PERMISSION_GRANTED) {
            Intent callIntent = new Intent(Intent.ACTION_CALL, Uri.parse("tel:" + phoneNumber));
            startActivity(callIntent);
        } else {
            Toast.makeText(this, "CALL_PHONE permission denied.", Toast.LENGTH_SHORT).show();
        }
    }

    private void sendSmsToEmergencyContacts(String message) {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.SEND_SMS) != PackageManager.PERMISSION_GRANTED) {
            Toast.makeText(this, "SMS permission denied.", Toast.LENGTH_SHORT).show();
            return;
        }

        SmsManager smsManager = SmsManager.getDefault();
        for (String contact : emergencyContacts) {
            smsManager.sendTextMessage(contact, null, message, null, null);
        }
    }

    private boolean isInternetAvailable() {
        ConnectivityManager cm = (ConnectivityManager) getSystemService(Context.CONNECTIVITY_SERVICE);
        NetworkInfo info = cm != null ? cm.getActiveNetworkInfo() : null;
        return info != null && info.isConnected();
    }

    private double distance(double lat1, double lon1, double lat2, double lon2) {
        float[] result = new float[1];
        Location.distanceBetween(lat1, lon1, lat2, lon2, result);
        return result[0];
    }

    private void createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(CHANNEL_ID, "SOS Alerts", NotificationManager.IMPORTANCE_HIGH);
            channel.setDescription("Channel for SOS alert notifications");
            NotificationManager manager = getSystemService(NotificationManager.class);
            if (manager != null) manager.createNotificationChannel(channel);
        }
    }

    private void showSOSNotification(String message) {
        NotificationCompat.Builder builder = new NotificationCompat.Builder(this, CHANNEL_ID)
                .setSmallIcon(R.drawable.ic_sos)  // Ensure you have ic_sos in drawable
                .setContentTitle("🚨 Medical SOS Alert")
                .setContentText(message)
                .setPriority(NotificationCompat.PRIORITY_HIGH)
                .setAutoCancel(true);

        NotificationManager manager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
        if (manager != null) manager.notify(1, builder.build());
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions,
                                           @NonNull int[] grantResults) {
        if (requestCode == PERMISSION_REQUEST_CODE) {
            boolean allGranted = true;
            for (int res : grantResults) {
                if (res != PackageManager.PERMISSION_GRANTED) {
                    allGranted = false;
                    break;
                }
            }

            if (allGranted) {
                startLocationAndLoadContacts();
            } else {
                Toast.makeText(this, "Permissions denied. App functionality limited.", Toast.LENGTH_LONG).show();
                // Optionally, guide user to app settings:
                new AlertDialog.Builder(this)
                        .setTitle("Permissions Required")
                        .setMessage("Please enable permissions in app settings for full functionality.")
                        .setPositiveButton("Open Settings", (dialog, which) -> {
                            Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
                            Uri uri = Uri.fromParts("package", getPackageName(), null);
                            intent.setData(uri);
                            startActivity(intent);
                        })
                        .setNegativeButton("Cancel", null)
                        .show();
            }
        } else {
            super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (locationCallback != null) {
            fusedLocationClient.removeLocationUpdates(locationCallback);
        }
    }
}
