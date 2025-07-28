import javax.swing.*;
import java.awt.*;
import java.sql.*;
import java.time.LocalDate;
import java.util.*;
import java.io.*;
import java.security.GeneralSecurityException;

import com.google.api.services.sheets.v4.*;
import com.google.api.services.sheets.v4.model.*;
import com.google.api.client.googleapis.auth.oauth2.*;
import com.google.api.client.googleapis.javanet.*;
import com.google.api.client.json.jackson2.*;
import com.google.api.client.http.javanet.NetHttpTransport;

public class AplikasiKasAnggota {

   public static void main(String[] args) {
        Database.init();
        new LoginFrame();
    }

    // ========================== Database ==========================
   static class Database {
        public static Connection connect() {
            try {
                Class.forName("org.sqlite.JDBC");
                return DriverManager.getConnection("jdbc:sqlite:kas.db");
            } catch (Exception e) {
                e.printStackTrace();
                return null;
            }
        }

   public static void init() {
            try (Connection conn = connect(); Statement stmt = conn.createStatement()) {
                stmt.execute("CREATE TABLE IF NOT EXISTS anggota (id INTEGER PRIMARY KEY AUTOINCREMENT, nama TEXT, username TEXT UNIQUE, password TEXT)");
                stmt.execute("CREATE TABLE IF NOT EXISTS simpanan (id INTEGER PRIMARY KEY AUTOINCREMENT, anggota_id INTEGER, tanggal TEXT, jumlah INTEGER, FOREIGN KEY(anggota_id) REFERENCES anggota(id))");
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }

    // ========================== LoginFrame ==========================
   static class LoginFrame extends JFrame {
        public LoginFrame() {
            setTitle("Login");
            setSize(300, 150);
            setLayout(new GridLayout(3, 2));
            setDefaultCloseOperation(EXIT_ON_CLOSE);

   JTextField tfUser = new JTextField();
            JPasswordField tfPass = new JPasswordField();
            JButton btnLogin = new JButton("Login");
            JButton btnDaftar = new JButton("Daftar");

  add(new JLabel("Username"));
            add(tfUser);
            add(new JLabel("Password"));
            add(tfPass);
            add(btnLogin);
            add(btnDaftar);

   btnLogin.addActionListener(e -> {
                try (Connection conn = Database.connect()) {
                    PreparedStatement stmt = conn.prepareStatement("SELECT * FROM anggota WHERE username = ? AND password = ?");
                    stmt.setString(1, tfUser.getText());
                    stmt.setString(2, new String(tfPass.getPassword()));
                    ResultSet rs = stmt.executeQuery();
                    if (rs.next()) {
                        new MainFrame(rs.getInt("id"), rs.getString("nama"));
                        dispose();
                    } else {
                        JOptionPane.showMessageDialog(this, "Login gagal!");
                    }
                } catch (Exception ex) {
                    ex.printStackTrace();
                }
            });

   btnDaftar.addActionListener(e -> new RegisterFrame());

   setVisible(true);
        }
    }

    // ========================== RegisterFrame ==========================
   static class RegisterFrame extends JFrame {
        public RegisterFrame() {
            setTitle("Daftar");
            setSize(300, 200);
            setLayout(new GridLayout(4, 2));
            setDefaultCloseOperation(DISPOSE_ON_CLOSE);

   JTextField tfNama = new JTextField();
            JTextField tfUser = new JTextField();
            JPasswordField tfPass = new JPasswordField();
            JButton btnRegister = new JButton("Daftar");

   add(new JLabel("Nama"));
            add(tfNama);
            add(new JLabel("Username"));
            add(tfUser);
            add(new JLabel("Password"));
            add(tfPass);
            add(new JLabel());
            add(btnRegister);

  btnRegister.addActionListener(e -> {
                try (Connection conn = Database.connect()) {
                    PreparedStatement stmt = conn.prepareStatement("INSERT INTO anggota (nama, username, password) VALUES (?, ?, ?)");
                    stmt.setString(1, tfNama.getText());
                    stmt.setString(2, tfUser.getText());
                    stmt.setString(3, new String(tfPass.getPassword()));
                    stmt.executeUpdate();
                    JOptionPane.showMessageDialog(this, "Registrasi berhasil!");
                    dispose();
                } catch (SQLException ex) {
                    JOptionPane.showMessageDialog(this, "Username sudah digunakan!");
                }
            });

   setVisible(true);
        }
    }

    // ========================== MainFrame ==========================
   static class MainFrame extends JFrame {
        int anggotaId;
        String nama;

   public MainFrame(int anggotaId, String nama) {
            this.anggotaId = anggotaId;
            this.nama = nama;

   setTitle("Dashboard - " + nama);
            setSize(400, 300);
            setLayout(new BorderLayout());
            setDefaultCloseOperation(EXIT_ON_CLOSE);

   JTextArea area = new JTextArea();
            JTextField tfJumlah = new JTextField();
            JButton btnSimpan = new JButton("Simpan");

   JPanel panel = new JPanel(new BorderLayout());
            panel.add(new JLabel("Jumlah Simpanan"), BorderLayout.NORTH);
            panel.add(tfJumlah, BorderLayout.CENTER);
            panel.add(btnSimpan, BorderLayout.EAST);

   add(panel, BorderLayout.NORTH);
            add(new JScrollPane(area), BorderLayout.CENTER);

  btnSimpan.addActionListener(e -> {
                try (Connection conn = Database.connect()) {
                    String tanggal = LocalDate.now().toString();
                    int jumlah = Integer.parseInt(tfJumlah.getText());

  PreparedStatement stmt = conn.prepareStatement("INSERT INTO simpanan (anggota_id, tanggal, jumlah) VALUES (?, ?, ?)");
                    stmt.setInt(1, anggotaId);
                    stmt.setString(2, tanggal);
                    stmt.setInt(3, jumlah);
                    stmt.executeUpdate();

  GoogleSheetService.kirimData(tanggal, nama, jumlah);
                    loadRiwayat(area);
                } catch (Exception ex) {
                    ex.printStackTrace();
                }
            });  loadRiwayat(area);
            setVisible(true);
        }

 void loadRiwayat(JTextArea area) {
            area.setText("");
            try (Connection conn = Database.connect()) {
                PreparedStatement stmt = conn.prepareStatement("SELECT tanggal, jumlah FROM simpanan WHERE anggota_id = ?");
                stmt.setInt(1, anggotaId);
                ResultSet rs = stmt.executeQuery();
                while (rs.next()) {
                    area.append(rs.getString("tanggal") + " - Rp" + rs.getInt("jumlah") + "\n");
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    // ========================== GoogleSheetService ==========================
static class GoogleSheetService {
        private static final String APPLICATION_NAME = "AplikasiKasJava";
        private static final String SPREADSHEET_ID = "ISI_ID_SPREADSHEET_KAMU"; // GANTI DENGAN ID SPREADSHEET KAMU
        private static final String SHEET_NAME = "Sheet1";
        private static Sheets sheetsService;

 public static Sheets getSheetsService() throws IOException, GeneralSecurityException {
            if (sheetsService == null) {
                GoogleCredential credential = GoogleCredential.fromStream(new FileInputStream("credentials.json"))
                        .createScoped(Collections.singletonList("https://www.googleapis.com/auth/spreadsheets"));
                sheetsService = new Sheets.Builder(new NetHttpTransport(), JacksonFactory.getDefaultInstance(), credential)
                        .setApplicationName(APPLICATION_NAME)
                        .build();
            }
            return sheetsService;
        }

 public static void kirimData(String tanggal, String nama, int jumlah) {
            try {
                Sheets service = getSheetsService();
                List<Object> row = Arrays.asList(tanggal, nama, jumlah);
                List<List<Object>> values = Collections.singletonList(row);
                ValueRange body = new ValueRange().setValues(values);
                service.spreadsheets().values()
                        .append(SPREADSHEET_ID, SHEET_NAME + "!A:C", body)
                        .setValueInputOption("RAW")
                        .execute();
            } catch (Exception e) {
                System.out.println("Gagal kirim ke Google Sheet: " + e.getMessage());
            }
        }
    }
}
