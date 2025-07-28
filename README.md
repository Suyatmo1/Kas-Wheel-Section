import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.sql.*;
import java.time.LocalDate;

public class AplikasiKasGabung {
    public static void main(String[] args) {
        Database.init(); // inisialisasi database
        new LoginFrame();
    }
}

// === DATABASE SETUP ===
class Database {
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
            String anggotaTable = "CREATE TABLE IF NOT EXISTS anggota (" +
                    "id INTEGER PRIMARY KEY AUTOINCREMENT," +
                    "nama TEXT," +
                    "username TEXT UNIQUE," +
                    "password TEXT)";
            stmt.execute(anggotaTable);

            String simpananTable = "CREATE TABLE IF NOT EXISTS simpanan (" +
                    "id INTEGER PRIMARY KEY AUTOINCREMENT," +
                    "anggota_id INTEGER," +
                    "tanggal TEXT," +
                    "jumlah INTEGER," +
                    "FOREIGN KEY(anggota_id) REFERENCES anggota(id))";
            stmt.execute(simpananTable);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}

// === LOGIN FRAME ===
class LoginFrame extends JFrame {
    public LoginFrame() {
        setTitle("Login Anggota");
        setSize(300, 150);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLayout(new GridLayout(3, 2));

        JTextField tfUsername = new JTextField();
        JPasswordField tfPassword = new JPasswordField();
        JButton btnLogin = new JButton("Login");
        JButton btnDaftar = new JButton("Daftar");

        add(new JLabel("Username"));
        add(tfUsername);
        add(new JLabel("Password"));
        add(tfPassword);
        add(btnLogin);
        add(btnDaftar);

        btnLogin.addActionListener(e -> {
            try (Connection conn = Database.connect()) {
                String sql = "SELECT * FROM anggota WHERE username = ? AND password = ?";
                PreparedStatement stmt = conn.prepareStatement(sql);
                stmt.setString(1, tfUsername.getText());
                stmt.setString(2, new String(tfPassword.getPassword()));
                ResultSet rs = stmt.executeQuery();
                if (rs.next()) {
                    int id = rs.getInt("id");
                    String nama = rs.getString("nama");
                    new MainFrame(id, nama);
                    dispose();
                } else {
                    JOptionPane.showMessageDialog(this, "Login gagal!");
                }
            } catch (SQLException ex) {
                ex.printStackTrace();
            }
        });

        btnDaftar.addActionListener(e -> new RegisterFrame());

        setVisible(true);
    }
}

// === REGISTER FRAME ===
class RegisterFrame extends JFrame {
    public RegisterFrame() {
        setTitle("Registrasi Anggota");
        setSize(300, 200);
        setDefaultCloseOperation(JFrame.DISPOSE_ON_CLOSE);
        setLayout(new GridLayout(4, 2));

        JTextField tfNama = new JTextField();
        JTextField tfUsername = new JTextField();
        JPasswordField tfPassword = new JPasswordField();
        JButton btnRegister = new JButton("Daftar");

        add(new JLabel("Nama"));
        add(tfNama);
        add(new JLabel("Username"));
        add(tfUsername);
        add(new JLabel("Password"));
        add(tfPassword);
        add(new JLabel(""));
        add(btnRegister);

        btnRegister.addActionListener(e -> {
            try (Connection conn = Database.connect()) {
                String sql = "INSERT INTO anggota (nama, username, password) VALUES (?, ?, ?)";
                PreparedStatement stmt = conn.prepareStatement(sql);
                stmt.setString(1, tfNama.getText());
                stmt.setString(2, tfUsername.getText());
                stmt.setString(3, new String(tfPassword.getPassword()));
                stmt.executeUpdate();
                JOptionPane.showMessageDialog(this, "Registrasi berhasil!");
                dispose();
            } catch (SQLException ex) {
                JOptionPane.showMessageDialog(this, "Username sudah digunakan.");
            }
        });

        setVisible(true);
    }
}

// === MAIN FRAME ===
class MainFrame extends JFrame {
    int anggotaId;
    String nama;

    public MainFrame(int anggotaId, String nama) {
        this.anggotaId = anggotaId;
        this.nama = nama;

        setTitle("Dashboard - " + nama);
        setSize(400, 300);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLayout(new BorderLayout());

        JTextArea taRiwayat = new JTextArea();
        JScrollPane scroll = new JScrollPane(taRiwayat);
        JTextField tfJumlah = new JTextField();
        JButton btnSimpan = new JButton("Simpan");

        JPanel panelInput = new JPanel(new BorderLayout());
        panelInput.add(new JLabel("Jumlah Simpanan"), BorderLayout.NORTH);
        panelInput.add(tfJumlah, BorderLayout.CENTER);
        panelInput.add(btnSimpan, BorderLayout.EAST);

        add(panelInput, BorderLayout.NORTH);
        add(scroll, BorderLayout.CENTER);

        btnSimpan.addActionListener(e -> {
            try (Connection conn = Database.connect()) {
                String sql = "INSERT INTO simpanan (anggota_id, tanggal, jumlah) VALUES (?, ?, ?)";
                PreparedStatement stmt = conn.prepareStatement(sql);
                stmt.setInt(1, anggotaId);
                stmt.setString(2, LocalDate.now().toString());
                stmt.setInt(3, Integer.parseInt(tfJumlah.getText()));
                stmt.executeUpdate();
                loadRiwayat(taRiwayat);
            } catch (Exception ex) {
                JOptionPane.showMessageDialog(this, "Gagal menyimpan.");
            }
        });

        loadRiwayat(taRiwayat);
        setVisible(true);
    }

    void loadRiwayat(JTextArea area) {
        area.setText("");
        try (Connection conn = Database.connect()) {
            String sql = "SELECT tanggal, jumlah FROM simpanan WHERE anggota_id = ?";
            PreparedStatement stmt = conn.prepareStatement(sql);
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
