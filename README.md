<!--?xml version="1.0" encoding="utf-8"?-->
<android.support.constraint.constraintlayout
xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:app="http://schemas.android.com/apk/res-auto"
xmlns:tools="http://schemas.android.com/tools"
android:layout_width="match_parent"
android:layout_height="match_parent"
tools:context="ru.maxfad.myraspisanie.MainActivity">

<relativelayout
android:layout_width="match_parent"
android:layout_height="match_parent">

<button
android:id="@+id/button11"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_alignparentstart="true"
android:layout_alignparenttop="true"
android:layout_marginstart="15dp"
android:layout_margintop="134dp"
android:text="@string/button11"
tools:layout_editor_absolutex="7dp"
tools:layout_editor_absolutey="131dp"></button>

<button
android:id="@+id/button10"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_alignbaseline="@+id/button11"
android:layout_alignbottom="@+id/button11"
android:layout_marginstart="22dp"
android:layout_toendof="@+id/button11"
android:text="@string/button10"
tools:layout_editor_absolutex="0dp"
tools:layout_editor_absolutey="0dp"></button>

<button
android:id="@+id/button9"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_alignbaseline="@+id/button10"
android:layout_alignbottom="@+id/button10"
android:layout_marginstart="21dp"
android:layout_toendof="@+id/button10"
android:text="@string/button9"></button>

<button
android:id="@+id/button8"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_alignstart="@+id/button11"
android:layout_below="@+id/button11"
android:layout_margintop="38dp"
android:text="@string/button8"></button>

<button
android:id="@+id/button7"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_alignbaseline="@+id/button8"
android:layout_alignbottom="@+id/button8"
android:layout_alignstart="@+id/button10"
android:text="@string/button7"></button>

<button
android:id="@+id/button6"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_alignbaseline="@+id/button7"
android:layout_alignbottom="@+id/button7"
android:layout_alignstart="@+id/button9"
android:text="@string/button6"></button>

</relativelayout>

</android.support.constraint.constraintlayout>
