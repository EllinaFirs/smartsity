import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.net.Uri;
import android.os.Bundle;
import android.text.TextUtils;
import android.util.Base64;
import android.widget.Button;
import android.widget.EditText;
import android.widget.ImageView;
import android.widget.Toast;

import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;
import androidx.appcompat.app.AppCompatActivity;

import java.io.ByteArrayOutputStream;
import java.io.InputStream;

public class AddNewsActivity extends AppCompatActivity {
    private EditText etTitle, etContent;
    private Button btnSave, btnPickImage;
    private ImageView imgPreview;
    private DBHelper db;
    private String imageBase64;

    private final ActivityResultLauncher<String> pickImageLauncher =
            registerForActivityResult(new ActivityResultContracts.GetContent(), this::onImagePicked);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_add_news);

        // Инициализация тулбара и обработка стрелки назад
        com.google.android.material.appbar.MaterialToolbar toolbar = findViewById(R.id.toolbar);
        if (toolbar != null) {
            setSupportActionBar(toolbar);
            if (getSupportActionBar() != null) {
                getSupportActionBar().setDisplayHomeAsUpEnabled(true);
                getSupportActionBar().setTitle("Добавить новость");
            }
            toolbar.setNavigationOnClickListener(v -> finish());
        }

        db = new DBHelper(this);

        etTitle = findViewById(R.id.etTitle);
        etContent = findViewById(R.id.etContent);
        btnSave = findViewById(R.id.btnSaveNews);
        btnPickImage = findViewById(R.id.btnPickImage);
        imgPreview = findViewById(R.id.imgPreview);

        btnPickImage.setOnClickListener(v -> pickImageLauncher.launch("image/*"));

        btnSave.setOnClickListener(v -> {
            String t = etTitle.getText().toString().trim();
            String c = etContent.getText().toString().trim();
            if (TextUtils.isEmpty(t) || TextUtils.isEmpty(c)) {
                Toast.makeText(AddNewsActivity.this, "Введите заголовок и текст", Toast.LENGTH_SHORT).show();
                return;
            }
            db.addNews(t, c, imageBase64);
            Toast.makeText(AddNewsActivity.this, "Новость добавлена", Toast.LENGTH_SHORT).show();
            finish();
        });
    }

    private void onImagePicked(Uri uri) {
        if (uri == null) return;
        try {
            InputStream is = getContentResolver().openInputStream(uri);
            if (is == null) return;
            Bitmap bmp = BitmapFactory.decodeStream(is);
            is.close();
            // Optionally compress to reduce size
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            bmp.compress(Bitmap.CompressFormat.JPEG, 80, baos);
            byte[] bytes = baos.toByteArray();
            imageBase64 = Base64.encodeToString(bytes, Base64.NO_WRAP);
            imgPreview.setImageBitmap(bmp);
            imgPreview.setVisibility(ImageView.VISIBLE);
        } catch (Exception e) {
            Toast.makeText(this, "Не удалось загрузить фото", Toast.LENGTH_SHORT).show();
        }
    }
}
import android.os.Bundle;
import android.text.TextUtils;
import android.widget.TextView;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;

import com.google.android.material.appbar.MaterialToolbar;
import com.google.android.material.button.MaterialButton;
import com.google.android.material.textfield.TextInputEditText;

public class AddServiceRequestActivity extends AppCompatActivity {
    private DBHelper db;
    private int serviceId;
    private String username;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_add_service_request);

        db = new DBHelper(this);

        MaterialToolbar toolbar = findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);
        toolbar.setNavigationOnClickListener(v -> finish());

        serviceId = getIntent().getIntExtra("service_id", -1);
        username = getIntent().getStringExtra("username");
        String serviceName = getIntent().getStringExtra("service_name");

        TextView tvServiceName = findViewById(R.id.tvServiceName);
        tvServiceName.setText("Услуга: " + serviceName);

        TextInputEditText etComment = findViewById(R.id.etComment);
        MaterialButton btnSend = findViewById(R.id.btnSendRequest);
        btnSend.setOnClickListener(v -> {
            String comment = etComment.getText() != null ? etComment.getText().toString().trim() : "";
            if (TextUtils.isEmpty(comment)) {
                Toast.makeText(AddServiceRequestActivity.this, "Введите комментарий", Toast.LENGTH_SHORT).show();
                return;
            }
            if (username == null) {
                Toast.makeText(AddServiceRequestActivity.this, "Неизвестный пользователь", Toast.LENGTH_SHORT).show();
                return;
            }
            db.addServiceRequest(username, serviceId, comment);
            Toast.makeText(AddServiceRequestActivity.this, "Заявка отправлена", Toast.LENGTH_SHORT).show();
            finish();
        });
    }
}
import android.content.Intent;
import android.os.Bundle;
import android.widget.AdapterView;
import android.widget.ArrayAdapter;
import android.widget.ListView;

import androidx.appcompat.app.AppCompatActivity;

import com.google.android.material.appbar.MaterialToolbar;

import java.util.ArrayList;
import java.util.List;

public class AdminComplaintsActivity extends AppCompatActivity {
    private DBHelper db;
    private ListView listView;
    private List<DBHelper.Complaint> complaints;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_admin_complaints);
        db = new DBHelper(this);

        MaterialToolbar toolbar = findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);
        toolbar.setNavigationOnClickListener(v -> finish());

        listView = findViewById(R.id.listComplaints);
        loadComplaints();

        listView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> parent, android.view.View view, int position, long id) {
                DBHelper.Complaint c = complaints.get(position);
                Intent i = new Intent(AdminComplaintsActivity.this, RespondComplaintActivity.class);
                i.putExtra("complaint_id", c.id);
                startActivity(i);
            }
        });
    }

    @Override
    protected void onResume() {
        super.onResume();
        loadComplaints();
    }

    private void loadComplaints() {
        complaints = db.getAllComplaints();
        List<String> lines = new ArrayList<>();
        for (DBHelper.Complaint c : complaints) {
            String statusLine = "Статус: " + mapStatus(c.status);
            String respLine = (c.response == null || c.response.isEmpty()) ? "" : ("\nОтвет: " + c.response);
            lines.add(c.username + ": " + c.text + "\n" + statusLine + respLine);
        }
        ArrayAdapter<String> adapter = new ArrayAdapter<>(this, android.R.layout.simple_list_item_1, lines);
        listView.setAdapter(adapter);
    }

    private String mapStatus(String s) {
        if (s == null || s.isEmpty()) return "Открыта";
        String v = s.toLowerCase();
        if (v.equals("open")) return "Открыта";
        if (v.equals("answered")) return "Отвечена";
        return s;
    }
}
import android.content.Context;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

import androidx.annotation.NonNull;
import androidx.recyclerview.widget.RecyclerView;

import com.google.android.material.button.MaterialButton;
import com.google.android.material.chip.Chip;
import com.google.android.material.color.MaterialColors;

import java.util.List;

public class AdminRequestAdapter extends RecyclerView.Adapter<AdminRequestAdapter.ViewHolder> {
    private final Context context;
    private final List<DBHelper.ServiceRequest> requests;
    private final DBHelper db;

    public AdminRequestAdapter(Context context, List<DBHelper.ServiceRequest> requests, DBHelper db) {
        this.context = context;
        this.requests = requests;
        this.db = db;
    }

    @NonNull
    @Override
    public ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View v = LayoutInflater.from(context).inflate(R.layout.item_admin_request, parent, false);
        return new ViewHolder(v);
    }

    @Override
    public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
        DBHelper.ServiceRequest r = requests.get(position);
        String serviceName = db.getServiceNameById(r.serviceId);
        holder.tvServiceName.setText(serviceName);
        holder.tvUsername.setText("Пользователь: " + r.username);
        holder.tvComment.setText("Комментарий: " + r.comment);
        holder.chipStatus.setText(mapStatus(r.status));
        // Цветовая индикация статуса
        int primary = MaterialColors.getColor(holder.chipStatus, com.google.android.material.R.attr.colorPrimary);
        int error = MaterialColors.getColor(holder.chipStatus, com.google.android.material.R.attr.colorError);
        int neutral = 0xFF9E9E9E; // серый
        switch (r.status) {
            case "approved":
                holder.chipStatus.setChipBackgroundColorResource(android.R.color.transparent);
                holder.chipStatus.setChipBackgroundColor(android.content.res.ColorStateList.valueOf(primary));
                holder.chipStatus.setTextColor(0xFFFFFFFF);
                break;
            case "rejected":
                holder.chipStatus.setChipBackgroundColor(android.content.res.ColorStateList.valueOf(error));
                holder.chipStatus.setTextColor(0xFFFFFFFF);
                break;
            default: // submitted
                holder.chipStatus.setChipBackgroundColor(android.content.res.ColorStateList.valueOf(neutral));
                holder.chipStatus.setTextColor(0xFFFFFFFF);
                break;
        }

        boolean submitted = "submitted".equals(r.status);
        holder.btnApprove.setEnabled(submitted);
        holder.btnReject.setEnabled(submitted);
        holder.btnApprove.setAlpha(submitted ? 1f : 0.6f);
        holder.btnReject.setAlpha(submitted ? 1f : 0.6f);

        holder.btnApprove.setOnClickListener(view -> {
            if (!"submitted".equals(r.status)) return;
            db.updateServiceRequestStatus(r.id, "approved");
            r.status = "approved";
            notifyItemChanged(position);
        });
        holder.btnReject.setOnClickListener(view -> {
            if (!"submitted".equals(r.status)) return;
            db.updateServiceRequestStatus(r.id, "rejected");
            r.status = "rejected";
            notifyItemChanged(position);
        });
    }

    @Override
    public int getItemCount() {
        return requests.size();
    }

    private String mapStatus(String status) {
        if (status == null) return "";
        switch (status) {
            case "submitted":
                return "Отправлено";
            case "approved":
                return "Одобрено";
            case "rejected":
                return "Отклонено";
            default:
                return status;
        }
    }

    static class ViewHolder extends RecyclerView.ViewHolder {
        TextView tvServiceName;
        TextView tvUsername;
        TextView tvComment;
        Chip chipStatus;
        MaterialButton btnApprove;
        MaterialButton btnReject;

        ViewHolder(@NonNull View itemView) {
            super(itemView);
            tvServiceName = itemView.findViewById(R.id.tvServiceName);
            tvUsername = itemView.findViewById(R.id.tvUsername);
            tvComment = itemView.findViewById(R.id.tvComment);
            chipStatus = itemView.findViewById(R.id.chipStatus);
            btnApprove = itemView.findViewById(R.id.btnApprove);
            btnReject = itemView.findViewById(R.id.btnReject);
        }
    }
}
import android.os.Bundle;
import android.view.View;

import androidx.annotation.Nullable;
import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import com.google.android.material.appbar.MaterialToolbar;

import java.util.List;

public class AdminRequestsActivity extends AppCompatActivity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_admin_requests);

        MaterialToolbar toolbar = findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);
        toolbar.setNavigationOnClickListener(v -> finish());

        RecyclerView recycler = findViewById(R.id.recyclerAdminRequests);
        recycler.setLayoutManager(new LinearLayoutManager(this));

        DBHelper db = new DBHelper(this);
        List<DBHelper.ServiceRequest> all = db.getAllServiceRequests();
        AdminRequestAdapter adapter = new AdminRequestAdapter(this, all, db);
        recycler.setAdapter(adapter);
    }
}
import android.os.Bundle;
import android.text.TextUtils;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;

import com.google.android.material.appbar.MaterialToolbar;

public class ComplaintActivity extends AppCompatActivity {
    private EditText etText;
    private Button btnSend;
    private DBHelper db;
    private String username;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_complaint);
        db = new DBHelper(this);
        username = getIntent().getStringExtra("username");

        MaterialToolbar toolbar = findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);
        if (getSupportActionBar() != null) {
            getSupportActionBar().setDisplayHomeAsUpEnabled(true);
            getSupportActionBar().setTitle("Отправить жалобу");
        }
        toolbar.setNavigationOnClickListener(v -> finish());

        etText = findViewById(R.id.etComplaint);
        btnSend = findViewById(R.id.btnSendComplaint);

        btnSend.setOnClickListener(v -> {
            String text = etText.getText().toString().trim();
            if (TextUtils.isEmpty(text)) {
                Toast.makeText(ComplaintActivity.this, "Введите текст жалобы", Toast.LENGTH_SHORT).show();
                return;
            }
            db.addComplaint(username, text);
            Toast.makeText(ComplaintActivity.this, "Жалоба отправлена", Toast.LENGTH_SHORT).show();
            finish();
        });
    }
}
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

import androidx.annotation.NonNull;
import androidx.recyclerview.widget.RecyclerView;

import java.util.List;

public class ComplaintAdapter extends RecyclerView.Adapter<ComplaintAdapter.ComplaintVH> {
    private final List<DBHelper.Complaint> items;

    public ComplaintAdapter(List<DBHelper.Complaint> items) {
        this.items = items;
    }

    @NonNull
    @Override
    public ComplaintVH onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View v = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_complaint, parent, false);
        return new ComplaintVH(v);
    }

    @Override
    public void onBindViewHolder(@NonNull ComplaintVH holder, int position) {
        DBHelper.Complaint c = items.get(position);
        holder.tvComplaintText.setText(c.text);
        String statusRu = mapStatus(c.status);
        holder.tvComplaintStatus.setText("Статус: " + statusRu);
        String resp = (c.response == null || c.response.isEmpty()) ? "Ответ пока не дан" : c.response;
        holder.tvComplaintResponse.setText("Ответ: " + resp);
    }

    @Override
    public int getItemCount() {
        return items.size();
    }

    static class ComplaintVH extends RecyclerView.ViewHolder {
        TextView tvComplaintText;
        TextView tvComplaintStatus;
        TextView tvComplaintResponse;
        ComplaintVH(@NonNull View itemView) {
            super(itemView);
            tvComplaintText = itemView.findViewById(R.id.tvComplaintText);
            tvComplaintStatus = itemView.findViewById(R.id.tvComplaintStatus);
            tvComplaintResponse = itemView.findViewById(R.id.tvComplaintResponse);
        }
    }

    private String mapStatus(String s) {
        if (s == null || s.isEmpty()) return "Открыта";
        String v = s.toLowerCase();
        if (v.equals("open")) return "Открыта";
        if (v.equals("answered")) return "Отвечена";
        return s;
    }
}
import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;

import java.util.ArrayList;
import java.util.List;

public class DBHelper extends SQLiteOpenHelper {
    private static final String DB_NAME = "civic_app.db";
    private static final int DB_VERSION = 3;

    public DBHelper(Context context) {
        super(context, DB_NAME, null, DB_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE, password TEXT, role TEXT)");
        db.execSQL("CREATE TABLE news (id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT, content TEXT, ts INTEGER, image_base64 TEXT)");
        db.execSQL("CREATE TABLE complaints (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT, text TEXT, ts INTEGER, response TEXT, status TEXT)");
        db.execSQL("CREATE TABLE services (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, description TEXT, requirements TEXT)");
        db.execSQL("CREATE TABLE service_requests (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT, service_id INTEGER, comment TEXT, ts INTEGER, status TEXT)");

        // seed users
        ContentValues admin = new ContentValues();
        admin.put("username", "admin");
        admin.put("password", "admin");
        admin.put("role", "admin");
        db.insert("users", null, admin);

        ContentValues user = new ContentValues();
        user.put("username", "user");
        user.put("password", "user");
        user.put("role", "user");
        db.insert("users", null, user);

        // seed sample news
        ContentValues n1 = new ContentValues();
        n1.put("title", "Добро пожаловать");
        n1.put("content", "Приложение для граждан и администрации запущено");
        n1.put("ts", System.currentTimeMillis());
        n1.put("image_base64", (String) null);
        db.insert("news", null, n1);

        // seed services catalog
        ContentValues s1 = new ContentValues();
        s1.put("name", "Регистрация по месту жительства");
        s1.put("description", "Подача заявления на регистрацию или изменение места жительства.");
        s1.put("requirements", "Паспорт, заявление, договор найма или свидетельство собственности.");
        db.insert("services", null, s1);

        ContentValues s2 = new ContentValues();
        s2.put("name", "Выдача справки");
        s2.put("description", "Получение официальной справки из администрации (о составе семьи, проживании и т.п.).");
        s2.put("requirements", "Паспорт, заявление, при необходимости — подтверждающие документы.");
        db.insert("services", null, s2);

        ContentValues s3 = new ContentValues();
        s3.put("name", "Запись на прием к специалисту");
        s3.put("description", "Онлайн-запись на консультацию в профильном отделе администрации.");
        s3.put("requirements", "Паспорт, краткое описание вопроса.");
        db.insert("services", null, s3);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        if (oldVersion < 2) {
            db.execSQL("ALTER TABLE news ADD COLUMN image_base64 TEXT");
        }
        if (oldVersion < 3) {
            db.execSQL("CREATE TABLE IF NOT EXISTS services (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, description TEXT, requirements TEXT)");
            db.execSQL("CREATE TABLE IF NOT EXISTS service_requests (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT, service_id INTEGER, comment TEXT, ts INTEGER, status TEXT)");
            // optional seed to ensure catalog exists after upgrade
            Cursor cur = db.query("services", new String[]{"id"}, null, null, null, null, null);
            try {
                if (!cur.moveToFirst()) {
                    ContentValues s1 = new ContentValues();
                    s1.put("name", "Регистрация по месту жительства");
                    s1.put("description", "Подача заявления на регистрацию или изменение места жительства.");
                    s1.put("requirements", "Паспорт, заявление, договор найма или свидетельство собственности.");
                    db.insert("services", null, s1);

                    ContentValues s2 = new ContentValues();
                    s2.put("name", "Выдача справки");
                    s2.put("description", "Получение официальной справки из администрации (о составе семьи, проживании и т.п.).");
                    s2.put("requirements", "Паспорт, заявление, при необходимости — подтверждающие документы.");
                    db.insert("services", null, s2);
                }
            } finally {
                cur.close();
            }
        }
    }

    public String getRoleIfValid(String username, String password) {
        SQLiteDatabase db = getReadableDatabase();
        Cursor c = db.query("users", new String[]{"role"}, "username=? AND password=?", new String[]{username, password}, null, null, null);
        try {
            if (c.moveToFirst()) {
                return c.getString(0);
            }
            return null;
        } finally {
            c.close();
        }
    }

    public List<NewsItem> getAllNews() {
        SQLiteDatabase db = getReadableDatabase();
        Cursor c = db.query("news", new String[]{"id", "title", "content", "ts", "image_base64"}, null, null, null, null, "ts DESC");
        List<NewsItem> list = new ArrayList<>();
        try {
            while (c.moveToNext()) {
                NewsItem item = new NewsItem();
                item.id = c.getInt(0);
                item.title = c.getString(1);
                item.content = c.getString(2);
                item.ts = c.getLong(3);
                item.imageBase64 = c.getString(4);
                list.add(item);
            }
        } finally {
            c.close();
        }
        return list;
    }

    public void addNews(String title, String content) {
        addNews(title, content, null);
    }

    public void addNews(String title, String content, String imageBase64) {
        SQLiteDatabase db = getWritableDatabase();
        ContentValues cv = new ContentValues();
        cv.put("title", title);
        cv.put("content", content);
        cv.put("ts", System.currentTimeMillis());
        cv.put("image_base64", imageBase64);
        db.insert("news", null, cv);
    }

    public List<Complaint> getComplaintsByUser(String username) {
        SQLiteDatabase db = getReadableDatabase();
        Cursor c = db.query("complaints", new String[]{"id", "username", "text", "ts", "response", "status"}, "username=?", new String[]{username}, null, null, "ts DESC");
        List<Complaint> list = new ArrayList<>();
        try {
            while (c.moveToNext()) {
                Complaint comp = new Complaint();
                comp.id = c.getInt(0);
                comp.username = c.getString(1);
                comp.text = c.getString(2);
                comp.ts = c.getLong(3);
                comp.response = c.getString(4);
                comp.status = c.getString(5);
                list.add(comp);
            }
        } finally {
            c.close();
        }
        return list;
    }

    public void addComplaint(String username, String text) {
        SQLiteDatabase db = getWritableDatabase();
        ContentValues cv = new ContentValues();
        cv.put("username", username);
        cv.put("text", text);
        cv.put("ts", System.currentTimeMillis());
        cv.put("response", "");
        cv.put("status", "open");
        db.insert("complaints", null, cv);
    }

    public List<Complaint> getAllComplaints() {
        SQLiteDatabase db = getReadableDatabase();
        Cursor c = db.query("complaints", new String[]{"id", "username", "text", "ts", "response", "status"}, null, null, null, null, "ts DESC");
        List<Complaint> list = new ArrayList<>();
        try {
            while (c.moveToNext()) {
                Complaint comp = new Complaint();
                comp.id = c.getInt(0);
                comp.username = c.getString(1);
                comp.text = c.getString(2);
                comp.ts = c.getLong(3);
                comp.response = c.getString(4);
                comp.status = c.getString(5);
                list.add(comp);
            }
        } finally {
            c.close();
        }
        return list;
    }

    public void respondToComplaint(int id, String response) {
        SQLiteDatabase db = getWritableDatabase();
        ContentValues cv = new ContentValues();
        cv.put("response", response);
        cv.put("status", "answered");
        db.update("complaints", cv, "id=?", new String[]{String.valueOf(id)});
    }

    // Simple models
    public static class NewsItem {
        public int id;
        public String title;
        public String content;
        public long ts;
        public String imageBase64;
    }

    public static class Complaint {
        public int id;
        public String username;
        public String text;
        public long ts;
        public String response;
        public String status;
    }

    public List<Service> getAllServices() {
        SQLiteDatabase db = getReadableDatabase();
        Cursor c = db.query("services", new String[]{"id", "name", "description", "requirements"}, null, null, null, null, "name ASC");
        List<Service> list = new ArrayList<>();
        try {
            while (c.moveToNext()) {
                Service s = new Service();
                s.id = c.getInt(0);
                s.name = c.getString(1);
                s.description = c.getString(2);
                s.requirements = c.getString(3);
                list.add(s);
            }
        } finally {
            c.close();
        }
        return list;
    }

    public String getServiceNameById(int id) {
        SQLiteDatabase db = getReadableDatabase();
        Cursor c = db.query("services", new String[]{"name"}, "id=?", new String[]{String.valueOf(id)}, null, null, null);
        try {
            if (c.moveToFirst()) {
                return c.getString(0);
            }
            return String.valueOf(id);
        } finally {
            c.close();
        }
    }

    public void addServiceRequest(String username, int serviceId, String comment) {
        SQLiteDatabase db = getWritableDatabase();
        ContentValues cv = new ContentValues();
        cv.put("username", username);
        cv.put("service_id", serviceId);
        cv.put("comment", comment);
        cv.put("ts", System.currentTimeMillis());
        cv.put("status", "submitted");
        db.insert("service_requests", null, cv);
    }

    public List<ServiceRequest> getServiceRequestsByUser(String username) {
        SQLiteDatabase db = getReadableDatabase();
        Cursor c = db.query("service_requests", new String[]{"id", "username", "service_id", "comment", "ts", "status"}, "username=?", new String[]{username}, null, null, "ts DESC");
        List<ServiceRequest> list = new ArrayList<>();
        try {
            while (c.moveToNext()) {
                ServiceRequest r = new ServiceRequest();
                r.id = c.getInt(0);
                r.username = c.getString(1);
                r.serviceId = c.getInt(2);
                r.comment = c.getString(3);
                r.ts = c.getLong(4);
                r.status = c.getString(5);
                list.add(r);
            }
        } finally {
            c.close();
        }
        return list;
    }

    public void updateServiceRequestStatus(int id, String status) {
        SQLiteDatabase db = getWritableDatabase();
        ContentValues cv = new ContentValues();
        cv.put("status", status);
        db.update("service_requests", cv, "id=?", new String[]{String.valueOf(id)});
    }

    public List<ServiceRequest> getAllServiceRequests() {
        SQLiteDatabase db = getReadableDatabase();
        Cursor c = db.query("service_requests", new String[]{"id", "username", "service_id", "comment", "ts", "status"}, null, null, null, null, "ts DESC");
        List<ServiceRequest> list = new ArrayList<>();
        try {
            while (c.moveToNext()) {
                ServiceRequest r = new ServiceRequest();
                r.id = c.getInt(0);
                r.username = c.getString(1);
                r.serviceId = c.getInt(2);
                r.comment = c.getString(3);
                r.ts = c.getLong(4);
                r.status = c.getString(5);
                list.add(r);
            }
        } finally {
            c.close();
        }
        return list;
    }

    public static class Service {
        public int id;
        public String name;
        public String description;
        public String requirements;
    }

    public static class ServiceRequest {
        public int id;
        public String username;
        public int serviceId;
        public String comment;
        public long ts;
        public String status;
    }
}
import android.content.Intent;
import android.os.Bundle;
import android.text.TextUtils;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;

public class LoginActivity extends AppCompatActivity {
    private EditText usernameEt;
    private EditText passwordEt;
    private Button loginBtn;
    private DBHelper db;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);

        db = new DBHelper(this);

        usernameEt = findViewById(R.id.etUsername);
        passwordEt = findViewById(R.id.etPassword);
        loginBtn = findViewById(R.id.btnLogin);

        loginBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                String u = usernameEt.getText().toString().trim();
                String p = passwordEt.getText().toString().trim();
                if (TextUtils.isEmpty(u) || TextUtils.isEmpty(p)) {
                    Toast.makeText(LoginActivity.this, "Введите логин и пароль", Toast.LENGTH_SHORT).show();
                    return;
                }
                String role = db.getRoleIfValid(u, p);
                if (role == null) {
                    Toast.makeText(LoginActivity.this, "Неверные данные", Toast.LENGTH_SHORT).show();
                    return;
                }
                Intent intent = new Intent(LoginActivity.this, MainActivity.class);
                intent.putExtra("username", u);
                intent.putExtra("role", role);
                startActivity(intent);
                finish();
            }
        });
    }
}
import android.content.Intent;
import android.os.Bundle;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.widget.LinearLayout;

import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import com.google.android.material.appbar.MaterialToolbar;
import com.google.android.material.floatingactionbutton.FloatingActionButton;
import com.google.android.material.button.MaterialButton;

import java.util.List;

public class MainActivity extends AppCompatActivity {
    private DBHelper db;
    private String username;
    private String role;
    private RecyclerView recyclerView;
    private FloatingActionButton fabPrimary;
    private MaterialButton btnAdminComplaints, btnSendComplaint, btnMyComplaints, btnServices, btnMyRequests, btnAdminRequests;
    private LinearLayout rowComplaints, rowServicesRequests, rowAdminRequests;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        db = new DBHelper(this);
        username = getIntent().getStringExtra("username");
        role = getIntent().getStringExtra("role");

        MaterialToolbar toolbar = findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        recyclerView = findViewById(R.id.recyclerNews);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        fabPrimary = findViewById(R.id.fabPrimary);
        btnAdminComplaints = findViewById(R.id.btnAdminComplaints);
        btnSendComplaint = findViewById(R.id.btnSendComplaint);
        btnMyComplaints = findViewById(R.id.btnMyComplaints);
        btnServices = findViewById(R.id.btnServices);
        btnMyRequests = findViewById(R.id.btnMyRequests);
        btnAdminRequests = findViewById(R.id.btnAdminRequests);

        rowComplaints = findViewById(R.id.rowComplaints);
        rowServicesRequests = findViewById(R.id.rowServicesRequests);
        rowAdminRequests = findViewById(R.id.rowAdminRequests);

        setupButtons();
        loadNews();
    }

    @Override
    protected void onResume() {
        super.onResume();
        loadNews();
    }

    private void setupButtons() {
        if ("admin".equals(role)) {
            fabPrimary.setContentDescription("Добавить новость");
            fabPrimary.setOnClickListener(v -> startActivity(new Intent(MainActivity.this, AddNewsActivity.class)));
            btnAdminComplaints.setVisibility(View.VISIBLE);
            btnAdminComplaints.setOnClickListener(v -> startActivity(new Intent(MainActivity.this, AdminComplaintsActivity.class)));
            btnSendComplaint.setVisibility(View.GONE);
            btnMyComplaints.setVisibility(View.GONE);
            // Админ не отправляет заявки: скрываем целую строку услуг/заявок, чтобы не было лишнего зазора
            rowServicesRequests.setVisibility(View.GONE);

            // Показать управление заявками (строка видна)
            rowAdminRequests.setVisibility(View.VISIBLE);
            btnAdminRequests.setVisibility(View.VISIBLE);
            btnAdminRequests.setOnClickListener(v -> {
                Intent i = new Intent(MainActivity.this, AdminRequestsActivity.class);
                startActivity(i);
            });
        } else {
            fabPrimary.setContentDescription("Отправить жалобу");
            fabPrimary.setOnClickListener(v -> {
                Intent i = new Intent(MainActivity.this, ComplaintActivity.class);
                i.putExtra("username", username);
                startActivity(i);
            });
            btnAdminComplaints.setVisibility(View.GONE);
            btnSendComplaint.setVisibility(View.GONE);
            btnMyComplaints.setVisibility(View.VISIBLE);
            btnMyComplaints.setOnClickListener(v -> {
                Intent i = new Intent(MainActivity.this, MyComplaintsActivity.class);
                i.putExtra("username", username);
                startActivity(i);
            });

            rowAdminRequests.setVisibility(View.GONE);
            btnAdminRequests.setVisibility(View.GONE);

            rowServicesRequests.setVisibility(View.VISIBLE);
            btnServices.setVisibility(View.VISIBLE);
            btnServices.setOnClickListener(v -> {
                Intent i = new Intent(MainActivity.this, ServicesActivity.class);
                i.putExtra("username", username);
                i.putExtra("role", role);
                startActivity(i);
            });
            btnMyRequests.setVisibility(View.VISIBLE);
            btnMyRequests.setOnClickListener(v -> {
                Intent i = new Intent(MainActivity.this, MyRequestsActivity.class);
                i.putExtra("username", username);
                startActivity(i);
            });
        }
    }

    private void loadNews() {
        List<DBHelper.NewsItem> items = db.getAllNews();
        recyclerView.setAdapter(new NewsAdapter(items));
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.menu_main, menu);
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        if (item.getItemId() == R.id.action_logout) {
            Intent i = new Intent(MainActivity.this, LoginActivity.class);
            i.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_NEW_TASK);
            startActivity(i);
            finish();
            return true;
        }
        return super.onOptionsItemSelected(item);
    }
}
