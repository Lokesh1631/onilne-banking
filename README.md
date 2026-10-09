import javax.swing.*;
import javax.swing.border.EmptyBorder;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.math.BigDecimal;
import java.sql.*;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.Base64;

/**
 * Educational desktop banking demo using Java Swing + JDBC + MySQL.
 * Do not use for real banking or production deployment.
 */
public class BankingApp {
    private static final String DB_URL = System.getenv().getOrDefault(
            "BANK_DB_URL", "jdbc:mysql://localhost:3306/online_banking?useSSL=false&serverTimezone=UTC");
    private static final String DB_USER = System.getenv().getOrDefault("BANK_DB_USER", "root");
    private static final String DB_PASSWORD = System.getenv().getOrDefault("BANK_DB_PASSWORD", "change_me");

    private final JFrame frame = new JFrame("Online Banking System");
    private long currentUserId;
    private String currentUserName;
    private String currentAccountNo;

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            try { UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName()); }
            catch (Exception ignored) {}
            new BankingApp().showLogin();
        });
    }

    private Connection connect() throws SQLException {
        return DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
    }

    private void configureFrame(String title, int width, int height) {
        frame.setTitle(title);
        frame.setSize(width, height);
        frame.setMinimumSize(new Dimension(700, 480));
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setLocationRelativeTo(null);
    }

    private JPanel page(String title, String subtitle) {
        JPanel p = new JPanel(new BorderLayout(12, 12));
        p.setBorder(new EmptyBorder(20, 24, 20, 24));
        p.setBackground(new Color(245, 248, 252));
        JPanel header = new JPanel(new BorderLayout());
        header.setOpaque(false);
        JLabel h = new JLabel(title);
        h.setFont(new Font("SansSerif", Font.BOLD, 25));
        h.setForeground(new Color(16, 72, 135));
        JLabel sub = new JLabel(subtitle);
        sub.setForeground(new Color(90, 105, 125));
        JPanel texts = new JPanel(new GridLayout(2, 1, 0, 5));
        texts.setOpaque(false);
        texts.add(h); texts.add(sub);
        header.add(texts, BorderLayout.WEST);
        p.add(header, BorderLayout.NORTH);
        return p;
    }

    private JButton button(String text) {
        JButton b = new JButton(text);
        b.setFocusPainted(false);
        b.setFont(new Font("SansSerif", Font.BOLD, 13));
        b.setBackground(new Color(20, 105, 210));
        b.setForeground(Color.WHITE);
        b.setBorder(new EmptyBorder(10, 16, 10, 16));
        return b;
    }

    private JTextField field(int columns) {
        JTextField f = new JTextField(columns);
        f.setFont(new Font("SansSerif", Font.PLAIN, 14));
        return f;
    }

    private void showLogin() {
        JPanel root = page("Online Banking System", "Secure • Simple • Java Swing desktop application");
        JPanel card = new JPanel(new GridBagLayout());
        card.setBackground(Color.WHITE);
        card.setBorder(BorderFactory.createCompoundBorder(
                BorderFactory.createLineBorder(new Color(220, 228, 238)),
                new EmptyBorder(24, 30, 24, 30)));
        GridBagConstraints c = new GridBagConstraints();
        c.insets = new Insets(9, 8, 9, 8);
        c.fill = GridBagConstraints.HORIZONTAL;
        c.gridx = 0; c.gridy = 0;
        card.add(new JLabel("User ID:"), c);
        JTextField user = field(20);
        c.gridx = 1; card.add(user, c);
        c.gridx = 0; c.gridy++;
        card.add(new JLabel("Password:"), c);
        JPasswordField pass = new JPasswordField(20);
        c.gridx = 1; card.add(pass, c);
        JButton login = button("Login");
        JButton register = new JButton("Create Account");
        JPanel actions = new JPanel(new FlowLayout(FlowLayout.LEFT, 10, 0));
        actions.setOpaque(false); actions.add(login); actions.add(register);
        c.gridx = 1; c.gridy++; card.add(actions, c);
        JLabel note = new JLabel("Demo application — never enter real banking credentials.");
        note.setForeground(new Color(170, 65, 45));
        c.gridx = 0; c.gridy++; c.gridwidth = 2; card.add(note, c);
        root.add(card, BorderLayout.CENTER);

        login.addActionListener(e -> doLogin(user.getText().trim(), new String(pass.getPassword())));
        pass.addActionListener(e -> doLogin(user.getText().trim(), new String(pass.getPassword())));
        register.addActionListener(e -> showRegister());
        configureFrame("Online Banking System - Login", 720, 440);
        frame.setContentPane(root); frame.setVisible(true);
    }

    private void showRegister() {
        JPanel root = page("Create New Account", "Register a demo customer account");
        JPanel form = new JPanel(new GridBagLayout());
        form.setBackground(Color.WHITE);
        form.setBorder(new EmptyBorder(18, 24, 18, 24));
        JTextField name = field(22), user = field(22);
        JPasswordField pass = new JPasswordField(22);
        JTextField deposit = field(22); deposit.setText("1000.00");
        String[] labels = {"Full name:", "User ID:", "Password:", "Initial deposit (₹):"};
        Component[] inputs = {name, user, pass, deposit};
        GridBagConstraints c = new GridBagConstraints();
        c.insets = new Insets(9, 8, 9, 8); c.fill = GridBagConstraints.HORIZONTAL;
        for (int i = 0; i < labels.length; i++) {
            c.gridx = 0; c.gridy = i; form.add(new JLabel(labels[i]), c);
            c.gridx = 1; form.add(inputs[i], c);
        }
        JButton create = button("Register");
        JButton back = new JButton("Back to Login");
        JPanel actions = new JPanel(new FlowLayout(FlowLayout.RIGHT));
        actions.setOpaque(false); actions.add(back); actions.add(create);
        c.gridx = 0; c.gridy = labels.length; c.gridwidth = 2; form.add(actions, c);
        root.add(form, BorderLayout.CENTER);
        create.addActionListener(e -> {
            String nm = name.getText().trim(), uid = user.getText().trim();
            String pw = new String(pass.getPassword());
            try {
                BigDecimal amount = new BigDecimal(deposit.getText().trim());
                if (nm.isEmpty() || uid.isEmpty() || pw.length() < 8 || amount.signum() < 0) {
                    message("Enter all fields, use a password of at least 8 characters, and a non-negative deposit."); return;
                }
                try (Connection con = connect()) {
                    con.setAutoCommit(false);
                    try {
                        String account = String.format("%010d", Math.abs(new SecureRandom().nextLong() % 10_000_000_000L));
                        String sql = "INSERT INTO users(full_name,user_id,password_hash,account_no,balance) VALUES(?,?,?,?,?)";
                        try (PreparedStatement ps = con.prepareStatement(sql)) {
                            ps.setString(1, nm); ps.setString(2, uid); ps.setString(3, hashPassword(pw));
                            ps.setString(4, account); ps.setBigDecimal(5, amount); ps.executeUpdate();
                        }
                        con.commit();
                        message("Account created successfully. Your demo account number is " + account);
                        showLogin();
                    } catch (SQLException ex) { con.rollback(); throw ex; }
                    finally { con.setAutoCommit(true); }
                }
            } catch (NumberFormatException ex) { message("Initial deposit must be a valid number."); }
            catch (SQLException ex) { message("Registration failed: " + ex.getMessage()); }
        });
        back.addActionListener(e -> showLogin());
        configureFrame("Online Banking System - Register", 720, 480);
        frame.setContentPane(root); frame.setVisible(true);
    }

    private void doLogin(String user, String password) {
        if (user.isEmpty() || password.isEmpty()) { message("Enter your user ID and password."); return; }
        try (Connection con = connect();
             PreparedStatement ps = con.prepareStatement(
                     "SELECT id,full_name,account_no,password_hash FROM users WHERE user_id=?")) {
            ps.setString(1, user);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next() && constantTimeEquals(rs.getString("password_hash"), hashPassword(password))) {
                    currentUserId = rs.getLong("id");
                    currentUserName = rs.getString("full_name");
                    currentAccountNo = rs.getString("account_no");
                    showDashboard();
                } else message("Invalid user ID or password.");
            }
        } catch (SQLException ex) { message("Could not connect to MySQL. Check database setup in README.md.\n" + ex.getMessage()); }
    }

    private void showDashboard() {
        JPanel root = page("Welcome, " + currentUserName, "Account overview • " + currentAccountNo);
        JPanel body = new JPanel(new BorderLayout(12, 12)); body.setOpaque(false);
        JPanel balanceCard = new JPanel(new GridLayout(2, 1, 4, 4));
        balanceCard.setBackground(new Color(224, 239, 255));
        balanceCard.setBorder(new EmptyBorder(18, 20, 18, 20));
        JLabel balTitle = new JLabel("AVAILABLE BALANCE");
        balTitle.setForeground(new Color(35, 83, 135));
        JLabel balValue = new JLabel("Loading...");
        balValue.setFont(new Font("SansSerif", Font.BOLD, 30));
        balValue.setForeground(new Color(13, 67, 130));
        balanceCard.add(balTitle); balanceCard.add(balValue);
        body.add(balanceCard, BorderLayout.NORTH);

        JPanel actions = new JPanel(new GridLayout(2, 2, 12, 12)); actions.setOpaque(false);
        JButton deposit = button("＋  Deposit Money");
        JButton withdraw = button("−  Withdraw Money");
        JButton transfer = button("⇄  Transfer Funds");
        JButton history = button("▤  Transaction History");
        actions.add(deposit); actions.add(withdraw); actions.add(transfer); actions.add(history);
        body.add(actions, BorderLayout.CENTER);
        JButton refresh = new JButton("Refresh Balance");
        JButton logout = new JButton("Logout");
        JPanel bottom = new JPanel(new FlowLayout(FlowLayout.RIGHT)); bottom.setOpaque(false);
        bottom.add(refresh); bottom.add(logout); body.add(bottom, BorderLayout.SOUTH);
        root.add(body, BorderLayout.CENTER);

        deposit.addActionListener(e -> moneyDialog("Deposit", null));
        withdraw.addActionListener(e -> moneyDialog("Withdraw", null));
        transfer.addActionListener(e -> transferDialog());
        history.addActionListener(e -> historyDialog());
        refresh.addActionListener(e -> refreshBalance(balValue));
        logout.addActionListener(e -> { currentUserId = 0; showLogin(); });
        refreshBalance(balValue);
        configureFrame("Online Banking System - Dashboard", 780, 530);
        frame.setContentPane(root); frame.setVisible(true);
    }

    private void refreshBalance(JLabel target) {
        try (Connection con = connect();
             PreparedStatement ps = con.prepareStatement("SELECT balance FROM users WHERE id=?")) {
            ps.setLong(1, currentUserId);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) target.setText("₹ " + rs.getBigDecimal(1).setScale(2, java.math.RoundingMode.HALF_UP));
            }
        } catch (SQLException ex) { target.setText("Unavailable"); }
    }

    private void moneyDialog(String type, String unused) {
        JTextField amountField = field(16);
        JTextField remarks = field(16); remarks.setText(type + " transaction");
        JPanel p = new JPanel(new GridLayout(0, 2, 8, 8));
        p.add(new JLabel("Amount (₹):")); p.add(amountField);
        p.add(new JLabel("Remarks:")); p.add(remarks);
        if (JOptionPane.showConfirmDialog(frame, p, type + " Money", JOptionPane.OK_CANCEL_OPTION) != JOptionPane.OK_OPTION) return;
        try {
            BigDecimal amount = new BigDecimal(amountField.getText().trim());
            if (amount.signum() <= 0) { message("Amount must be greater than zero."); return; }
            try (Connection con = connect()) {
                con.setAutoCommit(false);
                try {
                    BigDecimal balance;
                    try (PreparedStatement lock = con.prepareStatement("SELECT balance FROM users WHERE id=? FOR UPDATE")) {
                        lock.setLong(1, currentUserId);
                        try (ResultSet rs = lock.executeQuery()) {
                            if (!rs.next()) throw new SQLException("Account not found.");
                            balance = rs.getBigDecimal(1);
                        }
                    }
                    BigDecimal newBalance = type.equals("Deposit") ? balance.add(amount) : balance.subtract(amount);
                    if (newBalance.signum() < 0) { con.rollback(); message("Insufficient balance."); return; }
                    try (PreparedStatement up = con.prepareStatement("UPDATE users SET balance=? WHERE id=?")) {
                        up.setBigDecimal(1, newBalance); up.setLong(2, currentUserId); up.executeUpdate();
                    }
                    insertTransaction(con, currentUserId, type.toUpperCase(), amount, remarks.getText().trim());
                    con.commit();
                    message(type + " successful. New balance: ₹" + newBalance);
                } catch (SQLException ex) { con.rollback(); throw ex; }
                finally { con.setAutoCommit(true); }
            }
        } catch (NumberFormatException ex) { message("Enter a valid amount."); }
        catch (SQLException ex) { message("Transaction failed: " + ex.getMessage()); }
    }

    private void transferDialog() {
        JTextField receiver = field(16), amountField = field(16), remarks = field(16);
        JPanel p = new JPanel(new GridLayout(0, 2, 8, 8));
        p.add(new JLabel("Recipient account no.:")); p.add(receiver);
        p.add(new JLabel("Amount (₹):")); p.add(amountField);
        p.add(new JLabel("Remarks:")); p.add(remarks);
        if (JOptionPane.showConfirmDialog(frame, p, "Transfer Funds", JOptionPane.OK_CANCEL_OPTION) != JOptionPane.OK_OPTION) return;
        try {
            BigDecimal amount = new BigDecimal(amountField.getText().trim());
            String recipient = receiver.getText().trim();
            if (amount.signum() <= 0 || recipient.isEmpty()) { message("Enter a recipient and a positive amount."); return; }
            try (Connection con = connect()) {
                con.setAutoCommit(false);
                try {
                    long recipientId; BigDecimal senderBalance;
                    try (PreparedStatement ps = con.prepareStatement("SELECT id,balance FROM users WHERE id=? FOR UPDATE")) {
                        ps.setLong(1, currentUserId);
                        try (ResultSet rs = ps.executeQuery()) {
                            if (!rs.next()) throw new SQLException("Sender account not found.");
                            senderBalance = rs.getBigDecimal("balance");
                        }
                    }
                    try (PreparedStatement ps = con.prepareStatement("SELECT id FROM users WHERE account_no=? FOR UPDATE")) {
                        ps.setString(1, recipient);
                        try (ResultSet rs = ps.executeQuery()) {
                            if (!rs.next()) { con.rollback(); message("Recipient account not found."); return; }
                            recipientId = rs.getLong(1);
                        }
                    }
                    if (recipientId == currentUserId) { con.rollback(); message("Cannot transfer to your own account."); return; }
                    if (senderBalance.compareTo(amount) < 0) { con.rollback(); message("Insufficient balance."); return; }
                    try (PreparedStatement ps = con.prepareStatement("UPDATE users SET balance=balance-? WHERE id=?")) {
                        ps.setBigDecimal(1, amount); ps.setLong(2, currentUserId); ps.executeUpdate();
                    }
                    try (PreparedStatement ps = con.prepareStatement("UPDATE users SET balance=balance+? WHERE id=?")) {
                        ps.setBigDecimal(1, amount); ps.setLong(2, recipientId); ps.executeUpdate();
                    }
                    insertTransaction(con, currentUserId, "TRANSFER_OUT", amount, "To " + recipient + " - " + remarks.getText().trim());
                    insertTransaction(con, recipientId, "TRANSFER_IN", amount, "From " + currentAccountNo);
                    con.commit(); message("Transfer successful: ₹" + amount + " sent.");
                } catch (SQLException ex) { con.rollback(); throw ex; }
                finally { con.setAutoCommit(true); }
            }
        } catch (NumberFormatException ex) { message("Enter a valid amount."); }
        catch (SQLException ex) { message("Transfer failed: " + ex.getMessage()); }
    }

    private void insertTransaction(Connection con, long userId, String type, BigDecimal amount, String remarks) throws SQLException {
        try (PreparedStatement ps = con.prepareStatement(
                "INSERT INTO transactions(user_id,transaction_type,amount,remarks) VALUES(?,?,?,?)")) {
            ps.setLong(1, userId); ps.setString(2, type); ps.setBigDecimal(3, amount); ps.setString(4, remarks); ps.executeUpdate();
        }
    }

    private void historyDialog() {
        String[] cols = {"Date / Time", "Type", "Amount (₹)", "Remarks"};
        DefaultTableModel model = new DefaultTableModel(cols, 0) {
            public boolean isCellEditable(int row, int column) { return false; }
        };
        try (Connection con = connect();
             PreparedStatement ps = con.prepareStatement(
                     "SELECT created_at,transaction_type,amount,remarks FROM transactions WHERE user_id=? ORDER BY id DESC LIMIT 100")) {
            ps.setLong(1, currentUserId);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) model.addRow(new Object[]{rs.getTimestamp(1), rs.getString(2), rs.getBigDecimal(3), rs.getString(4)});
            }
        } catch (SQLException ex) { message("Could not load transaction history: " + ex.getMessage()); return; }
        JTable table = new JTable(model); table.setRowHeight(24);
        JScrollPane scroll = new JScrollPane(table); scroll.setPreferredSize(new Dimension(720, 320));
        JOptionPane.showMessageDialog(frame, scroll, "Transaction History", JOptionPane.PLAIN_MESSAGE);
    }

    // PBKDF2-style storage should be used in production. This demo uses SHA-256 only to keep
    // the source dependency-free; replace with BCrypt/Argon2/PBKDF2 for real authentication.
    private String hashPassword(String password) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            return Base64.getEncoder().encodeToString(md.digest(password.getBytes(java.nio.charset.StandardCharsets.UTF_8)));
        } catch (Exception ex) { throw new IllegalStateException(ex); }
    }

    private boolean constantTimeEquals(String a, String b) {
        if (a == null || b == null) return false;
        return MessageDigest.isEqual(a.getBytes(java.nio.charset.StandardCharsets.UTF_8),
                                     b.getBytes(java.nio.charset.StandardCharsets.UTF_8));
    }

    private void message(String text) { JOptionPane.showMessageDialog(frame, text, "Online Banking System", JOptionPane.INFORMATION_MESSAGE); }
}

