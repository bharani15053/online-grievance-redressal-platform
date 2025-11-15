package oopsproject;

import javax.swing.*;
import javax.swing.event.*;
import java.awt.*;
import java.awt.event.*;
import java.io.*;
import java.util.*;
import java.util.stream.*;
import java.util.List;

/**
 * Simple Online Grievance Redressal Platform using Java Swing.
 * Single-file application. Saves/loads grievances via serialization.
 */
public class Grievanceplatform extends JFrame {
    // Grievance model
    static class Grievance implements Serializable {
        private static final long serialVersionUID = 1L;
        private static int counter = 1;

        int id;
        String name;
        String description;
        String status; // "Pending", "In Progress", "Resolved"

        Grievance(String name, String description) {
            this.id = counter++;
            this.name = name;
            this.description = description;
            this.status = "Pending";
        }

        @Override
        public String toString() {
            return String.format("ID:%d | %s | %s", id, name, status);
        }
    }

    // UI components and data
    private DefaultListModel<Grievance> listModel = new DefaultListModel<>();
    private JList<Grievance> grievanceList = new JList<>(listModel);
    private JTextArea detailsArea = new JTextArea(10, 30);
    private JTextField nameField = new JTextField();
    private JTextArea descArea = new JTextArea(5, 30);
    private JComboBox<String> statusCombo = new JComboBox<>(new String[]{"Pending", "In Progress", "Resolved"});
    private JTextField searchField = new JTextField(20);

    public Grievanceplatform() {
        super("Online Grievance Redressal Platform");
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setSize(900, 600);
        setLocationRelativeTo(null);

        setupUI();
        attachListeners();
    }

    private void setupUI() {
        // Left panel: list and controls
        JPanel left = new JPanel(new BorderLayout(8, 8));
        left.setBorder(BorderFactory.createEmptyBorder(8,8,8,8));

        JLabel listLabel = new JLabel("Grievances");
        listLabel.setFont(listLabel.getFont().deriveFont(Font.BOLD, 14f));
        left.add(listLabel, BorderLayout.NORTH);

        grievanceList.setSelectionMode(ListSelectionModel.SINGLE_SELECTION);
        grievanceList.setFixedCellWidth(320);
        grievanceList.setCellRenderer(new DefaultListCellRenderer() {
            @Override
            public Component getListCellRendererComponent(JList<?> list, Object value, int index,
                                                          boolean isSelected, boolean cellHasFocus) {
                Component c = super.getListCellRendererComponent(list, value, index, isSelected, cellHasFocus);
                if (value instanceof Grievance) {
                    setText(((Grievance) value).toString());
                }
                return c;
            }
        });
        JScrollPane listScroll = new JScrollPane(grievanceList);
        left.add(listScroll, BorderLayout.CENTER);

        JPanel leftButtons = new JPanel(new FlowLayout(FlowLayout.LEFT));
        JButton btnDelete = new JButton("Delete");
        JButton btnSave = new JButton("Save...");
        JButton btnLoad = new JButton("Load...");
        leftButtons.add(btnDelete);
        leftButtons.add(btnSave);
        leftButtons.add(btnLoad);
        left.add(leftButtons, BorderLayout.SOUTH);

        // Right panel: detail and add form
        JPanel right = new JPanel();
        right.setBorder(BorderFactory.createEmptyBorder(8,8,8,8));
        right.setLayout(new BoxLayout(right, BoxLayout.Y_AXIS));

        // Search
        JPanel searchPanel = new JPanel(new FlowLayout(FlowLayout.LEFT));
        searchPanel.add(new JLabel("Search:"));
        searchPanel.add(searchField);
        JButton btnSearch = new JButton("Go");
        JButton btnClearSearch = new JButton("Clear");
        searchPanel.add(btnSearch);
        searchPanel.add(btnClearSearch);
        right.add(searchPanel);

        // Details area
        detailsArea.setEditable(false);
        detailsArea.setFont(new Font(Font.MONOSPACED, Font.PLAIN, 12));
        JScrollPane detailsScroll = new JScrollPane(detailsArea);
        detailsScroll.setBorder(BorderFactory.createTitledBorder("Details"));
        right.add(detailsScroll);

        // Status change
        JPanel statusPanel = new JPanel(new FlowLayout(FlowLayout.LEFT));
        statusPanel.add(new JLabel("Change status:"));
        statusPanel.add(statusCombo);
        JButton btnUpdateStatus = new JButton("Update");
        statusPanel.add(btnUpdateStatus);
        right.add(statusPanel);

        // Add grievance form
        JPanel addPanel = new JPanel();
        addPanel.setLayout(new BoxLayout(addPanel, BoxLayout.Y_AXIS));
        addPanel.setBorder(BorderFactory.createTitledBorder("Add New Grievance"));
        JPanel nameP = new JPanel(new BorderLayout(5,5));
        nameP.add(new JLabel("Name:"), BorderLayout.WEST);
        nameP.add(nameField, BorderLayout.CENTER);
        addPanel.add(nameP);

        JPanel descP = new JPanel(new BorderLayout(5,5));
        descP.add(new JLabel("Description:"), BorderLayout.NORTH);
        descArea.setLineWrap(true);
        descArea.setWrapStyleWord(true);
        JScrollPane descScroll = new JScrollPane(descArea);
        descP.add(descScroll, BorderLayout.CENTER);
        addPanel.add(descP);

        JButton btnAdd = new JButton("Add Grievance");
        JPanel addBtnP = new JPanel(new FlowLayout(FlowLayout.RIGHT));
        addBtnP.add(btnAdd);
        addPanel.add(addBtnP);

        right.add(addPanel);

        // Wire some external button references to listeners using actions stored in map
        // We'll store references to these buttons locally by searching the components.
        // For simplicity, add action listeners here:
        btnAdd.addActionListener(e -> addGrievance());
        btnDelete.addActionListener(e -> deleteSelected());
        btnUpdateStatus.addActionListener(e -> changeStatusOfSelected());
        btnSearch.addActionListener(e -> performSearch());
        btnClearSearch.addActionListener(e -> {
            searchField.setText("");
            refreshList();
        });
        btnSave.addActionListener(e -> saveToFile());
        btnLoad.addActionListener(e -> loadFromFile());

        // Combine left and right
        JSplitPane split = new JSplitPane(JSplitPane.HORIZONTAL_SPLIT, left, right);
        split.setDividerLocation(360);
        getContentPane().add(split, BorderLayout.CENTER);
    }

    private void attachListeners() {
        grievanceList.addListSelectionListener(e -> {
            if (!e.getValueIsAdjusting()) {
                Grievance g = grievanceList.getSelectedValue();
                showDetails(g);
            }
        });

        grievanceList.addMouseListener(new MouseAdapter() {
            public void mouseClicked(MouseEvent e) {
                if (e.getClickCount() == 2) {
                    Grievance g = grievanceList.getSelectedValue();
                    if (g != null) {
                        JOptionPane.showMessageDialog(Grievanceplatform.this,
                                formatFullDetails(g), "Grievance Full Details", JOptionPane.INFORMATION_MESSAGE);
                    }
                }
            }
        });
    }

    // Add new grievance
    private void addGrievance() {
        String name = nameField.getText().trim();
        String desc = descArea.getText().trim();

        if (name.isEmpty()) {
            JOptionPane.showMessageDialog(this, "Please enter a name/title.", "Validation", JOptionPane.WARNING_MESSAGE);
            return;
        }
        if (desc.isEmpty()) {
            JOptionPane.showMessageDialog(this, "Please enter a description.", "Validation", JOptionPane.WARNING_MESSAGE);
            return;
        }

        Grievance g = new Grievance(name, desc);
        listModel.addElement(g);
        nameField.setText("");
        descArea.setText("");
        grievanceList.setSelectedValue(g, true);
    }

    // Delete selected grievance
    private void deleteSelected() {
        Grievance selected = grievanceList.getSelectedValue();
        if (selected == null) {
            JOptionPane.showMessageDialog(this, "Select a grievance to delete.", "Info", JOptionPane.INFORMATION_MESSAGE);
            return;
        }
        int confirm = JOptionPane.showConfirmDialog(this,
                "Delete selected grievance?\n" + selected.toString(),
                "Confirm Delete", JOptionPane.YES_NO_OPTION);
        if (confirm == JOptionPane.YES_OPTION) {
            listModel.removeElement(selected);
            detailsArea.setText("");
        }
    }

    // Change status of selected
    private void changeStatusOfSelected() {
        Grievance selected = grievanceList.getSelectedValue();
        if (selected == null) {
            JOptionPane.showMessageDialog(this, "Select a grievance to update.", "Info", JOptionPane.INFORMATION_MESSAGE);
            return;
        }
        String newStatus = (String) statusCombo.getSelectedItem();
        selected.status = newStatus;
        grievanceList.repaint();
        showDetails(selected);
    }

    // Show details in text area
    private void showDetails(Grievance g) {
        if (g == null) {
            detailsArea.setText("");
            return;
        }
        detailsArea.setText(formatFullDetails(g));
    }

    private String formatFullDetails(Grievance g) {
        StringBuilder sb = new StringBuilder();
        sb.append("Grievance ID: ").append(g.id).append("\n");
        sb.append("Name/Title  : ").append(g.name).append("\n");
        sb.append("Status      : ").append(g.status).append("\n");
        sb.append("Description :\n");
        sb.append(g.description).append("\n");
        return sb.toString();
    }

    // Search (by id or substring in name/description)
    private void performSearch() {
        String q = searchField.getText().trim();
        if (q.isEmpty()) {
            refreshList();
            return;
        }
        // try parse id
        List<Grievance> results;
        try {
            int id = Integer.parseInt(q);
            results = Collections.list(listModel.elements())
                    .stream()
                    .filter(g -> g.id == id)
                    .collect(Collectors.toList());
        } catch (NumberFormatException ex) {
            String lower = q.toLowerCase();
            results = Collections.list(listModel.elements())
                    .stream()
                    .filter(g -> g.name.toLowerCase().contains(lower) || g.description.toLowerCase().contains(lower))
                    .collect(Collectors.toList());
        }

        DefaultListModel<Grievance> nm = new DefaultListModel<>();
        for (Grievance g : results) nm.addElement(g);
        grievanceList.setModel(nm);
    }

    // Refresh list to full model (after search clear)
    private void refreshList() {
        // rebuild a new DefaultListModel with all grievances from the backing model listModel
        DefaultListModel<Grievance> fresh = new DefaultListModel<>();
        for (int i = 0; i < listModel.size(); i++) {
            fresh.addElement(listModel.get(i));
        }
        grievanceList.setModel(fresh);
    }

    // Save to file (serialization)
    private void saveToFile() {
        JFileChooser chooser = new JFileChooser();
        chooser.setDialogTitle("Save grievances to file");
        int ret = chooser.showSaveDialog(this);
        if (ret != JFileChooser.APPROVE_OPTION) return;

        File f = chooser.getSelectedFile();
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(f))) {
            // persist current counter and list
            List<Grievance> all = Collections.list(listModel.elements());
            oos.writeObject(all);
            // save counter
            oos.writeInt(Grievance.counter);
            JOptionPane.showMessageDialog(this, "Saved " + all.size() + " grievances to " + f.getName());
        } catch (IOException ex) {
            ex.printStackTrace();
            JOptionPane.showMessageDialog(this, "Error saving file: " + ex.getMessage(), "Error", JOptionPane.ERROR_MESSAGE);
        }
    }

    // Load from file
    private void loadFromFile() {
        JFileChooser chooser = new JFileChooser();
        chooser.setDialogTitle("Load grievances from file");
        int ret = chooser.showOpenDialog(this);
        if (ret != JFileChooser.APPROVE_OPTION) return;

        File f = chooser.getSelectedFile();
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(f))) {
            Object obj = ois.readObject();
            if (!(obj instanceof List)) {
                JOptionPane.showMessageDialog(this, "File format not recognized.", "Error", JOptionPane.ERROR_MESSAGE);
                return;
            }
            @SuppressWarnings("unchecked")
            List<Grievance> loaded = (List<Grievance>) obj;
            int counterSaved = 1;
            try {
                counterSaved = ois.readInt();
            } catch (EOFException eof) {
                // older file may not contain counter; continue
            }
            listModel.clear();
            for (Grievance g : loaded) listModel.addElement(g);
            Grievance.counter = Math.max(Grievance.counter, counterSaved);
            refreshList();
            JOptionPane.showMessageDialog(this, "Loaded " + loaded.size() + " grievances from " + f.getName());
        } catch (Exception ex) {
            ex.printStackTrace();
            JOptionPane.showMessageDialog(this, "Error loading file: " + ex.getMessage(), "Error", JOptionPane.ERROR_MESSAGE);
        }
    }

    public static void main(String[] args) {
        // Preload with some sample data for convenience
        SwingUtilities.invokeLater(() -> {
            Grievanceplatform app = new Grievanceplatform();
            // sample entries
            app.listModel.addElement(new Grievance("Water supply issue", "No water supply since 3 days in block A."));
            app.listModel.addElement(new Grievance("Street light broken", "Street light near park not working at night."));
            app.listModel.addElement(new Grievance("Noise complaint", "Loud construction noises early morning."));
            app.setVisible(true);
        });
    }
}# online-grievance-redressal-platform
