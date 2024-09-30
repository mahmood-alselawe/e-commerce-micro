`public class Main {
    public static void main(String[] argv) throws IOException, UnsupportedAudioFileException {
        // Set Vosk logging level (optional)
        LibVosk.setLogLevel(LogLevel.DEBUG);

        // Use the correct path to the Arabic model
        String audioFilePath = "C:\\Users\\mahmad\\Desktop\\1السلام عليكم ورحمة ا.wav"; // Update this to your audio file path

        try (Model model = new Model("C:\\Users\\mahmad\\Desktop\\vosk-model-ar-0.22-linto-1.1.0");
             InputStream ais = AudioSystem
                     .getAudioInputStream(new BufferedInputStream(new FileInputStream(audioFilePath)));
             Recognizer recognizer = new Recognizer(model, 16000)) {

            // Process the audio file in chunks
            int nbytes;
            byte[] buffer = new byte[4096];
            while ((nbytes = ais.read(buffer)) >= 0) {
                if (recognizer.acceptWaveForm(buffer, nbytes)) {
                    System.out.println(recognizer.getResult());
                } else {
                    System.out.println(recognizer.getPartialResult());
                }
            }

            // Print the final recognition result
            System.out.println(recognizer.getFinalResult());
        } catch (UnsupportedAudioFileException e) {
            System.err.println("Audio file format not supported: " + e.getMessage());
        } catch (IOException e) {
            System.err.println("I/O error occurred: " + e.getMessage());
        } catch (Exception e) {
            System.err.println("An error occurred: " + e.getMessage());
        }
    }
}`
