public class Graph {
 
    //максимальное количество вершин в графе
    private final int VERTEX_MAX = 100;
    //массив для хранения вершин
    private Urbans[] urbansList;
    //текущее количество вершин в графе
    private int vertexCount;
 
    public int[][] getMatrix() {
        return matrix;
    }
 
    //матрица смежности
    private int[][] matrix;
 
 
 
    public Graph()
    {
        matrix = new int[VERTEX_MAX][VERTEX_MAX];
        //перед началом работы заполняем матрицу смежности нулями
        for(int i = 0; i < VERTEX_MAX; i++)
            for(int j = 0; j < VERTEX_MAX; j++)
                matrix[i][j] = 0;
        vertexCount = 0;
        urbansList = new Urbans[VERTEX_MAX];
    }
    public void addUrban(String label)
    {
        urbansList[vertexCount++] = new Urbans(label);
    }
 
    //добавление Urbans
    void addEdge(int begin, int end)
    {
        matrix[begin][end] = 1;
        matrix[end][begin] = 0;
    }
 
    int getSuccessor(int u)
    {
        for(int j = 0; j < vertexCount; j++)
            if(matrix[u][j] == 1 && urbansList[j].get_isVisited() == false)
                return j; //возвращает первую найденную вершину
        return -1; //таких вершин нет
    }
 
}
 
[size="1"][color="grey"][I]Добавлено через 20 секунд[/I][/color][/size]
public class Urbans {
 
    private String label;
    private boolean isVisited;
 
    public Urbans(String label)
    {
        this.label = label;
        isVisited = false;
    }
 
    public String getLabel() {
        return label;
    }
 
    public void setLabel(String label) {
        this.label = label;
    }
 
    public boolean get_isVisited() {
        return isVisited;
    }
 
    public void setVisited(boolean isVisited) {
        this.isVisited = isVisited;
    }
}
