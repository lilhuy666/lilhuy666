package com. pucescus.car;
import android.os.Bundle;
import android.widget.Button;
import android.widget.EditText;
import android.widget.ListView;
import androidx.appcompat.app.AppCompatActivity;
import java. util. ArrayList;
import java.util. List;
public class MainActivity extends AppCompatActivity {
private EditText taskInput;
private Button addTaskButton;
private ListView taskListView;
private List<String> tasks = nеw ArrayList<>();
private TaskAdapter taskAdapter;
@Override
protected void onCreate(Bundle savedInstanceState) {
super.onCreate(savedInstanceState);
setContentView(R.layout.activity_main);
taukInput = findviewById(R.id.toskinput):
addTaskButton = findViewById(R.ic.eddtoskbutton):
tasklistView s findViewById(R.id.tesklistview);
taskAdapter new TaskAdapter( comevt this, tasks);
tasklistView. setAdapter(taskAdapter);
addTaskButton.setünClickListener(v ->
( String task = taskInput.getText().toString();;
tasks.add(task):
taskAdapter.notifyDataSetChanged():
taskInput.setText(**);
F);
taskListView. setünItemClickListener((parent, view, position, id) -> { tasks.remove(position);
taskAdapter.notifyDataSetChanged();
}
}
}
это Мэйн активити

<LinearLayout xmlnstandraide" http://schenas.andreid.com/apk android: layout_width "match_parent" android: Layout_height="match_parent" android:ariontation "vertical">
<EditText
android: id="0+id/taskinput" android: Layout_width="match_parent" android: layout_height"srap_content" android:hint "Enter Task" />
<Button
android: id "+id/addtaskbutton" android: Layout_width "wrap_content" android: Layout_height "srap_content" android: text "Add Task" />
<ListView
android:id "@+id/tasklistview" android: Layout_width="match_marent" android: layout_height "Odp" android: Layout_weight "1"/>
</LinearLayout>
ето активити мейн

package
com. pucescus.car;
import |....
2 usages
public class TaskAdapter extends ArrayAdapter<String> {
1 usage
public TaskAdapter(Context context, List<String> tasks) ( super(context, (esurce 0, tasks):
@Override
public View getView(int position, View convertViem, ViewGroup parent) (
String task = getItem(position);
if (convertView = null) (
convertView = LayoutInflater.from(getContext()).inflate(android.R.layout.simple_list_item_1, parenttachToRoot false);
TextView taskText = convertView.findViewById(android.R.id.texti): taskText.setText(task);
return convertView:
}
}
и вот ещё хуйня
